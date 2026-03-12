# Add Discord File Sending

Adds `sendFile` implementation to the Discord channel so the agent can send file attachments (screenshots, generated code, PDFs, etc.) as Discord message attachments.

## Prerequisites

- Discord channel must already be set up (`src/channels/discord.ts` exists)
- Core file-sending infrastructure must be in place (`sendFile?` on Channel interface in `src/types.ts`, IPC file handling in `src/ipc.ts`)

## Implementation

Add the `sendFile` method to the `DiscordChannel` class in `src/channels/discord.ts`.

### Step 1: Add imports

Add `import fs from 'fs';` at the top of `src/channels/discord.ts` (if not already imported). Also add `GuildPremiumTier` to the discord.js import:

```typescript
import { GuildPremiumTier } from 'discord.js';
```

(Add it alongside the existing `TextChannel` import.)

### Step 2: Add `sendFile` method

Add this method to the `DiscordChannel` class, after the existing `sendMessage` method:

```typescript
async sendFile(jid: string, text: string, filePaths: string[]): Promise<void> {
  if (!this.client) {
    logger.warn('Discord client not initialized');
    return;
  }

  try {
    const channelId = jid.replace(/^dc:/, '');
    const channel = await this.client.channels.fetch(channelId);

    if (!channel || !('send' in channel)) {
      logger.warn({ jid }, 'Discord channel not found or not text-based');
      return;
    }

    const textChannel = channel as TextChannel;
    const MAX_FILES_PER_MESSAGE = 10;

    // Server boost tier → file upload limit in MB.
    // Source: https://support.discord.com/hc/en-us/articles/360028038352-Server-Boosting-FAQ
    const TIER_UPLOAD_LIMITS: Record<number, number> = {
      [GuildPremiumTier.None]: 25,
      [GuildPremiumTier.Tier1]: 25,
      [GuildPremiumTier.Tier2]: 50,
      [GuildPremiumTier.Tier3]: 100,
    };

    // Filter to existing files only (skip missing, log warning)
    const validFiles: string[] = [];
    for (const filePath of filePaths) {
      if (!fs.existsSync(filePath)) {
        logger.warn({ filePath }, 'Discord sendFile: file not found, skipping');
        continue;
      }
      validFiles.push(filePath);
    }

    if (validFiles.length === 0) {
      // No valid files — fall back to text-only
      if (text) {
        await this.sendMessage(jid, text);
      }
      return;
    }

    // Batch files (Discord allows max 10 per message)
    for (let i = 0; i < validFiles.length; i += MAX_FILES_PER_MESSAGE) {
      const batch = validFiles.slice(i, i + MAX_FILES_PER_MESSAGE);
      const isFirstBatch = i === 0;

      try {
        await textChannel.send({
          content: isFirstBatch && text ? text : undefined,
          files: batch,
        });
      } catch (err: any) {
        // Handle Discord file-too-large errors (40005 = RequestEntityTooLarge, 50045 = FileUploadedExceedsMaximumSize)
        if (err?.code === 40005 || err?.code === 50045) {
          const fileInfo = batch.map((f) => {
            const name = f.split('/').pop();
            try {
              const sizeMB = (fs.statSync(f).size / (1024 * 1024)).toFixed(1);
              return `${name} (${sizeMB}MB)`;
            } catch {
              return name;
            }
          }).join(', ');
          let errorMsg: string;

          if ('guild' in textChannel && textChannel.guild) {
            const tier = textChannel.guild.premiumTier;
            const limitMB = TIER_UPLOAD_LIMITS[tier] ?? 25;
            errorMsg = `File ${fileInfo} exceeds this server's ${limitMB}MB upload limit.`;
          } else {
            errorMsg = `File ${fileInfo} is too large to upload to Discord.`;
          }

          logger.warn({ jid, batch, code: err.code }, errorMsg);
          await this.sendMessage(jid, errorMsg);
          continue; // Continue with remaining batches
        }
        throw err; // Re-throw unexpected errors
      }
    }

    logger.info({ jid, fileCount: validFiles.length }, 'Discord file message sent');
  } catch (err) {
    logger.error({ jid, err }, 'Failed to send Discord file message');
  }
}
```

### Step 3: Add tests

Add these tests to `src/channels/discord.test.ts` in a new `describe('sendFile', ...)` block:

```typescript
describe('sendFile', () => {
  it('sends single file with text', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    await channel.connect();

    const mockChannel = {
      send: vi.fn().mockResolvedValue(undefined),
      sendTyping: vi.fn(),
    };
    currentClient().channels.fetch.mockResolvedValue(mockChannel);

    // Create a temp file
    const tmpFile = path.join(os.tmpdir(), `test-${Date.now()}.txt`);
    fs.writeFileSync(tmpFile, 'test content');

    try {
      await channel.sendFile('dc:1234567890123456', 'Here is the file', [tmpFile]);
      expect(mockChannel.send).toHaveBeenCalledWith({
        content: 'Here is the file',
        files: [tmpFile],
      });
    } finally {
      fs.unlinkSync(tmpFile);
    }
  });

  it('sends file without text', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    await channel.connect();

    const mockChannel = {
      send: vi.fn().mockResolvedValue(undefined),
      sendTyping: vi.fn(),
    };
    currentClient().channels.fetch.mockResolvedValue(mockChannel);

    const tmpFile = path.join(os.tmpdir(), `test-${Date.now()}.txt`);
    fs.writeFileSync(tmpFile, 'test');

    try {
      await channel.sendFile('dc:1234567890123456', '', [tmpFile]);
      expect(mockChannel.send).toHaveBeenCalledWith({
        content: undefined,
        files: [tmpFile],
      });
    } finally {
      fs.unlinkSync(tmpFile);
    }
  });

  it('skips missing files, falls back to text-only', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    await channel.connect();

    const mockChannel = {
      send: vi.fn().mockResolvedValue(undefined),
      sendTyping: vi.fn(),
    };
    currentClient().channels.fetch.mockResolvedValue(mockChannel);

    await channel.sendFile('dc:1234567890123456', 'fallback text', ['/nonexistent/file.png']);

    // Should fall back to sendMessage (text-only) since file doesn't exist
    // The send call comes from sendMessage's path
    expect(mockChannel.send).toHaveBeenCalledWith('fallback text');
  });

  it('does nothing when client is null', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    // Don't connect
    await channel.sendFile('dc:1234567890123456', 'text', ['/some/file.png']);
    // No error
  });

  it('sends error message when file exceeds server upload limit', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    await channel.connect();

    const discordError = new Error('Request entity too large');
    Object.assign(discordError, { code: 40005 });

    const mockChannel = {
      send: vi.fn().mockRejectedValue(discordError),
      sendTyping: vi.fn(),
      guild: { premiumTier: 0 }, // No boost = 25MB limit
    };
    currentClient().channels.fetch.mockResolvedValue(mockChannel);

    const tmpFile = path.join(os.tmpdir(), `test-large-${Date.now()}.bin`);
    fs.writeFileSync(tmpFile, 'x');

    try {
      await channel.sendFile('dc:1234567890123456', 'Here is a big file', [tmpFile]);

      // Should have sent an error message to the user via sendMessage
      const sendCalls = mockChannel.send.mock.calls;
      // Second call is from sendMessage fallback with the error text
      const errorCall = sendCalls.find(
        (call: any[]) => typeof call[0] === 'string' && call[0].includes('25MB upload limit')
      );
      expect(errorCall).toBeDefined();
    } finally {
      fs.unlinkSync(tmpFile);
    }
  });

  it('sends error message with boosted server limit for code 50045', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    await channel.connect();

    const discordError = new Error('File uploaded exceeds maximum size');
    Object.assign(discordError, { code: 50045 });

    const mockChannel = {
      send: vi.fn()
        .mockRejectedValueOnce(discordError)  // First call (file upload) fails
        .mockResolvedValue(undefined),          // Subsequent calls (error message) succeed
      sendTyping: vi.fn(),
      guild: { premiumTier: 2 }, // Tier 2 = 50MB limit
    };
    currentClient().channels.fetch.mockResolvedValue(mockChannel);

    const tmpFile = path.join(os.tmpdir(), `test-large-${Date.now()}.bin`);
    fs.writeFileSync(tmpFile, 'x');

    try {
      await channel.sendFile('dc:1234567890123456', '', [tmpFile]);

      // The error message should reference the 50MB limit
      const sendCalls = mockChannel.send.mock.calls;
      const errorCall = sendCalls.find(
        (call: any[]) => typeof call[0] === 'string' && call[0].includes('50MB upload limit')
      );
      expect(errorCall).toBeDefined();
    } finally {
      fs.unlinkSync(tmpFile);
    }
  });

  it('sends generic error when guild info is unavailable', async () => {
    const opts = createTestOpts();
    const channel = new DiscordChannel('test-token', opts);
    await channel.connect();

    const discordError = new Error('Request entity too large');
    Object.assign(discordError, { code: 40005 });

    const mockChannel = {
      send: vi.fn()
        .mockRejectedValueOnce(discordError)
        .mockResolvedValue(undefined),
      sendTyping: vi.fn(),
      // No guild property (e.g. DM channel)
    };
    currentClient().channels.fetch.mockResolvedValue(mockChannel);

    const tmpFile = path.join(os.tmpdir(), `test-large-${Date.now()}.bin`);
    fs.writeFileSync(tmpFile, 'x');

    try {
      await channel.sendFile('dc:1234567890123456', '', [tmpFile]);

      const sendCalls = mockChannel.send.mock.calls;
      const errorCall = sendCalls.find(
        (call: any[]) => typeof call[0] === 'string' && call[0].includes('too large to upload to Discord')
      );
      expect(errorCall).toBeDefined();
    } finally {
      fs.unlinkSync(tmpFile);
    }
  });
});
```

Note: Add `import fs from 'fs';`, `import os from 'os';`, and `import path from 'path';` at the top of the test file if not already present.

## Verification

1. `npm run build` — clean compile
2. `npx vitest run src/channels/discord.test.ts` — Discord tests pass
3. `./container/build.sh` — rebuild container
4. `cp container/agent-runner/src/*.ts data/sessions/discord_main/agent-runner-src/` — sync
5. `launchctl kickstart -k gui/$(id -u)/com.nanoclaw` — restart
6. In Discord: "Take a screenshot of example.com and send it to me" — should receive the image as a Discord attachment
