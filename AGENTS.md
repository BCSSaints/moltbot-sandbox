# Agent Instructions

Guidelines for AI agents working on this codebase.

## Project Overview

This is a Cloudflare Worker that runs [Moltbot](https://molt.bot/) in a Cloudflare Sandbox container. It provides:
- Proxying to the Moltbot gateway (web UI + WebSocket)
- Admin UI at `/_admin/` for device management
- API endpoints at `/api/*` for device pairing
- Debug endpoints at `/debug/*` for troubleshooting

**Note:** The CLI tool is still named `clawdbot` (upstream hasn't renamed yet), so CLI commands and internal config paths still use that name.

## Implementation Status

### Completion Status
- **Overall Progress**: 85% Complete
- - **Core Functionality**: ✅ Fully Implemented
  - - **Configuration Management**: ✅ Fully Implemented
    - - **Authentication/Authorization**: ✅ Fully Implemented
      - - **Testing**: 🟡 Partial (Unit tests complete, E2E tests in progress)
        - - **Documentation**: 🟡 Good (README complete, inline docs needed)
          -
          - ### Features Working
          - - ✅ OpenClaw gateway runs in Cloudflare Sandbox container
            - - ✅ Web UI control interface accessible via workers.dev domain
              - - ✅ WebSocket communication with control UI and clients
                - - ✅ Device pairing authentication system with admin approval
                  - - ✅ Cloudflare Access integration for admin UI protection
                    - - ✅ R2 persistent storage for config and paired devices
                      - - ✅ Environment variable management and secret handling
                        - - ✅ Browser automation (CDP) via cloudflare-browser skill
                          - - ✅ Multi-channel support (Telegram, Discord, Slack)
                            - - ✅ Admin UI with device management and gateway restart
                              - - ✅ Debug endpoints for troubleshooting
                                - - ✅ Local development mode (DEV_MODE)
                                  - - ✅ Cold start optimization with container lifecycle management
                                    -
                                    - ### Known Blockers & Challenges
                                    - 1. **WebSocket Issues in Local Dev**: `wrangler dev` has limitations with WebSocket proxying through the sandbox - HTTP requests work but WebSocket connections may fail. Full functionality requires deployment to Cloudflare.
                                      2.
                                      3. 2. **R2 Mount Limitations**:
                                         3.    - `rsync` compatibility requires `-r --no-times` flags (s3fs doesn't support timestamps)
                                               -    - Destructive operations on `/data/moltbot/` directly delete R2 backup data
                                                    -    - Mount status detection requires checking `mount | grep s3fs` instead of relying on API responses
                                                         -
                                                         - 3. **Process Status Detection**: The sandbox API's `proc.status` may not update immediately after process completion. Success detection requires checking for expected output rather than status flags.
                                                           4.
                                                           5. 4. **Cold Start Latency**: Initial container startup takes 1-2 minutes on first request. Subsequent requests are faster.
                                                              5.
                                                              6. 5. **Device List Command Delay**: CLI device list commands take 10-15 seconds due to WebSocket connection overhead.
                                                                 6.
                                                                 7. 6. **CLI Tool Naming**: The tool is still named `clawdbot` (upstream OpenClaw hasn't renamed yet), so CLI commands and internal config paths retain the old naming.
                                                                    7.
                                                                    8. ### Next Steps (Priority Order)
                                                                    9. 1. **Complete E2E Testing**: Add integration tests for gateway startup, device pairing workflow, and WebSocket communication
                                                                       2. 2. **Improve WebSocket Support**: Investigate workarounds for wrangler dev WebSocket proxying limitations or document deployment requirement
                                                                          3. 3. **Enhance R2 Sync Reliability**: Implement atomic sync operations and verification to prevent data loss scenarios
                                                                             4. 4. **Add Monitoring/Observability**: Implement container health checks, process monitoring, and operational logging
                                                                                5. 5. **Performance Optimization**: Profile cold start times and explore container initialization optimizations
                                                                                   6. 6. **Additional Chat Platform Support**: Integrate additional platforms (Teams, WhatsApp) with consistent DM policy handling
                                                                                      7. 7. **Upstream Updates**: Monitor OpenClaw upstream for tool/package naming changes and evaluate migration path
                                                                                         8.
                                                                                         9. ## Project Structure

```
src/
├── index.ts          # Main Hono app, route mounting
├── types.ts          # TypeScript type definitions
├── config.ts         # Constants (ports, timeouts, paths)
├── auth/             # Cloudflare Access authentication
│   ├── jwt.ts        # JWT verification
│   ├── jwks.ts       # JWKS fetching and caching
│   └── middleware.ts # Hono middleware for auth
├── gateway/          # Moltbot gateway management
│   ├── process.ts    # Process lifecycle (find, start)
│   ├── env.ts        # Environment variable building
│   ├── r2.ts         # R2 bucket mounting
│   ├── sync.ts       # R2 backup sync logic
│   └── utils.ts      # Shared utilities (waitForProcess)
├── routes/           # API route handlers
│   ├── api.ts        # /api/* endpoints (devices, gateway)
│   ├── admin.ts      # /_admin/* static file serving
│   └── debug.ts      # /debug/* endpoints
└── client/           # React admin UI (Vite)
    ├── App.tsx
    ├── api.ts        # API client
    └── pages/
```

## Key Patterns

### Environment Variables

- `DEV_MODE` - Skips CF Access auth AND bypasses device pairing (maps to `CLAWDBOT_DEV_MODE` for container)
- `DEBUG_ROUTES` - Enables `/debug/*` routes (disabled by default)
- See `src/types.ts` for full `MoltbotEnv` interface

### CLI Commands

When calling the moltbot CLI from the worker, always include `--url ws://localhost:18789`.
Note: The CLI is still named `clawdbot` until upstream renames it:
```typescript
sandbox.startProcess('clawdbot devices list --json --url ws://localhost:18789')
```

CLI commands take 10-15 seconds due to WebSocket connection overhead. Use `waitForProcess()` helper in `src/routes/api.ts`.

### Success Detection

The CLI outputs "Approved" (capital A). Use case-insensitive checks:
```typescript
stdout.toLowerCase().includes('approved')
```

## Commands

```bash
npm test              # Run tests (vitest)
npm run test:watch    # Run tests in watch mode
npm run build         # Build worker + client
npm run deploy        # Build and deploy to Cloudflare
npm run dev           # Vite dev server
npm run start         # wrangler dev (local worker)
npm run typecheck     # TypeScript check
```

## Testing

Tests use Vitest. Test files are colocated with source files (`*.test.ts`).

Current test coverage:
- `auth/jwt.test.ts` - JWT decoding and validation
- `auth/jwks.test.ts` - JWKS fetching and caching
- `auth/middleware.test.ts` - Auth middleware behavior
- `gateway/env.test.ts` - Environment variable building
- `gateway/process.test.ts` - Process finding logic
- `gateway/r2.test.ts` - R2 mounting logic

When adding new functionality, add corresponding tests.

## Code Style

- Use TypeScript strict mode
- Prefer explicit types over inference for function signatures
- Keep route handlers thin - extract logic to separate modules
- Use Hono's context methods (`c.json()`, `c.html()`) for responses

## Documentation

- `README.md` - User-facing documentation (setup, configuration, usage)
- `AGENTS.md` - This file, for AI agents

Development documentation goes in AGENTS.md, not README.md.

---

## Architecture

```
Browser
   │
   ▼
┌─────────────────────────────────────┐
│     Cloudflare Worker (index.ts)    │
│  - Starts Moltbot in sandbox        │
│  - Proxies HTTP/WebSocket requests  │
│  - Passes secrets as env vars       │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│     Cloudflare Sandbox Container    │
│  ┌───────────────────────────────┐  │
│  │     Moltbot Gateway           │  │
│  │  - Control UI on port 18789   │  │
│  │  - WebSocket RPC protocol     │  │
│  │  - Agent runtime              │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

### Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Worker that manages sandbox lifecycle and proxies requests |
| `Dockerfile` | Container image based on `cloudflare/sandbox` with Node 22 + Moltbot |
| `start-moltbot.sh` | Startup script that configures moltbot from env vars and launches gateway |
| `moltbot.json.template` | Default Moltbot configuration template |
| `wrangler.jsonc` | Cloudflare Worker + Container configuration |

## Local Development

```bash
npm install
cp .dev.vars.example .dev.vars
# Edit .dev.vars with your ANTHROPIC_API_KEY
npm run start
```

### Environment Variables

For local development, create `.dev.vars`:

```bash
ANTHROPIC_API_KEY=sk-ant-...
DEV_MODE=true           # Skips CF Access auth + device pairing
DEBUG_ROUTES=true       # Enables /debug/* routes
```

### WebSocket Limitations

Local development with `wrangler dev` has issues proxying WebSocket connections through the sandbox. HTTP requests work but WebSocket connections may fail. Deploy to Cloudflare for full functionality.

## Docker Image Caching

The Dockerfile includes a cache bust comment. When changing `moltbot.json.template` or `start-moltbot.sh`, bump the version:

```dockerfile
# Build cache bust: 2026-01-26-v10
```

## Gateway Configuration

Moltbot configuration is built at container startup:

1. `moltbot.json.template` is copied to `~/.clawdbot/clawdbot.json` (internal path unchanged)
2. `start-moltbot.sh` updates the config with values from environment variables
3. Gateway starts with `--allow-unconfigured` flag (skips onboarding wizard)

### Container Environment Variables

These are the env vars passed TO the container (internal names):

| Variable | Config Path | Notes |
|----------|-------------|-------|
| `ANTHROPIC_API_KEY` | (env var) | Moltbot reads directly from env |
| `CLAWDBOT_GATEWAY_TOKEN` | `--token` flag | Mapped from `MOLTBOT_GATEWAY_TOKEN` |
| `CLAWDBOT_DEV_MODE` | `controlUi.allowInsecureAuth` | Mapped from `DEV_MODE` |
| `TELEGRAM_BOT_TOKEN` | `channels.telegram.botToken` | |
| `DISCORD_BOT_TOKEN` | `channels.discord.token` | |
| `SLACK_BOT_TOKEN` | `channels.slack.botToken` | |
| `SLACK_APP_TOKEN` | `channels.slack.appToken` | |

## Moltbot Config Schema

Moltbot has strict config validation. Common gotchas:

- `agents.defaults.model` must be `{ "primary": "model/name" }` not a string
- `gateway.mode` must be `"local"` for headless operation
- No `webchat` channel - the Control UI is served automatically
- `gateway.bind` is not a config option - use `--bind` CLI flag

See [Moltbot docs](https://docs.molt.bot/gateway/configuration) for full schema.

## Common Tasks

### Adding a New API Endpoint

1. Add route handler in `src/routes/api.ts`
2. Add types if needed in `src/types.ts`
3. Update client API in `src/client/api.ts` if frontend needs it
4. Add tests

### Adding a New Environment Variable

1. Add to `MoltbotEnv` interface in `src/types.ts`
2. If passed to container, add to `buildEnvVars()` in `src/gateway/env.ts`
3. Update `.dev.vars.example`
4. Document in README.md secrets table

### Debugging

```bash
# View live logs
npx wrangler tail

# Check secrets
npx wrangler secret list
```

Enable debug routes with `DEBUG_ROUTES=true` and check `/debug/processes`.

## R2 Storage Notes

R2 is mounted via s3fs at `/data/moltbot`. Important gotchas:

- **rsync compatibility**: Use `rsync -r --no-times` instead of `rsync -a`. s3fs doesn't support setting timestamps, which causes rsync to fail with "Input/output error".

- **Mount checking**: Don't rely on `sandbox.mountBucket()` error messages to detect "already mounted" state. Instead, check `mount | grep s3fs` to verify the mount status.

- **Never delete R2 data**: The mount directory `/data/moltbot` IS the R2 bucket. Running `rm -rf /data/moltbot/*` will DELETE your backup data. Always check mount status before any destructive operations.
- 
## Implementation Roadmap

This section provides detailed guidance for implementing the 7 priority tasks identified in the Implementation Status section above.

### Task 1: Improve WebSocket Support (Foundation)

**Objective**: Make WebSocket connections reliable in local development and production environments.

**Files to Modify**:
- `src/index.ts` - Main worker handler
- - `vitest.config.ts` - Test configuration
  - - `src/routes/api.ts` - WebSocket endpoints
   
    - **Code Example: WebSocket Utility Layer**
   
    - Create `src/gateway/websocket-utils.ts`:
   
    - ```typescript
      import { Hono } from 'hono';
      import { upgradeWebSocket } from 'hono/cloudflare-workers';

      export interface WebSocketMessage {
        type: 'device-request' | 'device-response' | 'ping' | 'error';
        payload: unknown;
        timestamp: number;
      }

      export class WebSocketManager {
        private connections: Map<string, WebSocket> = new Map();
        private messageQueue: WebSocketMessage[] = [];
        private reconnectAttempts = 0;
        private maxReconnectAttempts = 5;

        async connect(url: string, clientId: string): Promise<WebSocket> {
          try {
            const ws = new WebSocket(url);

            ws.onopen = () => {
              console.log(`WebSocket connected for client: ${clientId}`);
              this.reconnectAttempts = 0;
              this.flushMessageQueue();
            };

            ws.onmessage = (event) => {
              this.handleMessage(event.data, clientId);
            };

            ws.onerror = (error) => {
              console.error(`WebSocket error for ${clientId}:`, error);
              this.handleError(clientId);
            };

            ws.onclose = () => {
              console.log(`WebSocket closed for ${clientId}`);
              this.connections.delete(clientId);
              this.attemptReconnect(url, clientId);
            };

            this.connections.set(clientId, ws);
            return ws;
          } catch (error) {
            console.error(`Failed to connect WebSocket for ${clientId}:`, error);
            throw error;
          }
        }

        private flushMessageQueue() {
          while (this.messageQueue.length > 0) {
            const msg = this.messageQueue.shift();
            if (msg) {
              this.broadcast(msg);
            }
          }
        }

        private attemptReconnect(url: string, clientId: string) {
          if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
            console.log(`Attempting to reconnect in ${delay}ms (attempt ${this.reconnectAttempts})`);
            setTimeout(() => this.connect(url, clientId), delay);
          } else {
            console.error(`Max reconnection attempts reached for ${clientId}`);
          }
        }

        private handleMessage(data: string, clientId: string) {
          try {
            const message: WebSocketMessage = JSON.parse(data);
            console.log(`Received message from ${clientId}:`, message.type);
          } catch (error) {
            console.error(`Failed to parse WebSocket message:`, error);
          }
        }

        private handleError(clientId: string) {
          // Implement error recovery logic
        }

        private broadcast(message: WebSocketMessage) {
          for (const [clientId, ws] of this.connections) {
            if (ws.readyState === WebSocket.OPEN) {
              ws.send(JSON.stringify(message));
            }
          }
        }

        send(clientId: string, message: WebSocketMessage) {
          const ws = this.connections.get(clientId);
          if (ws && ws.readyState === WebSocket.OPEN) {
            ws.send(JSON.stringify(message));
          } else {
            this.messageQueue.push(message);
          }
        }

        close(clientId: string) {
          const ws = this.connections.get(clientId);
          if (ws) {
            ws.close();
            this.connections.delete(clientId);
          }
        }
      }
      ```

      **Implementation Steps**:
      1. Create the WebSocket utility layer above
      2. 2. Add reconnection logic with exponential backoff
         3. 3. Implement message queuing for when connection is down
            4. 4. Update `src/index.ts` to use the new WebSocketManager
               5. 5. Add configuration option `WEBSOCKET_TIMEOUT` to environment
                  6. 6. Write tests in `src/gateway/websocket-utils.test.ts`
                    
                     7. **Testing**:
                     8. - Test connection establishment
                        - - Test message sending/receiving
                          - - Test reconnection logic
                            - - Test message queue behavior
                             
                              - ---

                              ### Task 2: Complete E2E Testing (Depends on Task 1)

                              **Objective**: Add comprehensive end-to-end tests for gateway startup, device pairing, and WebSocket communication.

                              **Files to Create**:
                              - `src/integration.test.ts` - Main integration test suite
                              - - `src/routes/api.integration.test.ts` - API integration tests
                                - - `src/client/integration.test.ts` - Client-side integration tests
                                 
                                  - **Code Template for `src/integration.test.ts`**:
                                 
                                  - ```typescript
                                    import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
                                    import { createMockEnv, createMockSandbox } from './test-utils';
                                    import type { MoltbotEnv } from './types';

                                    describe('E2E: Gateway Startup and Device Pairing', () => {
                                      let env: MoltbotEnv;

                                      beforeEach(() => {
                                        env = createMockEnv();
                                      });

                                      describe('Gateway Startup', () => {
                                        it('should start the moltbot gateway process', async () => {
                                          const { sandbox, startProcessMock } = createMockSandbox();
                                          // Setup env with sandbox

                                          // Assert gateway started
                                          expect(startProcessMock).toHaveBeenCalledWith(
                                            expect.objectContaining({
                                              image: expect.stringContaining('moltbot'),
                                            })
                                          );
                                        });

                                        it('should handle gateway startup timeout gracefully', async () => {
                                          // Test timeout handling
                                        });

                                        it('should recover from gateway crash', async () => {
                                          // Test recovery logic
                                        });
                                      });

                                      describe('Device Pairing Workflow', () => {
                                        it('should show pending devices in admin UI', async () => {
                                          // Test device listing
                                        });

                                        it('should approve a pending device', async () => {
                                          // Test device approval
                                        });

                                        it('should reject a pending device', async () => {
                                          // Test device rejection
                                        });

                                        it('should show paired devices list', async () => {
                                          // Test paired devices retrieval
                                        });
                                      });

                                      describe('WebSocket Communication', () => {
                                        it('should establish WebSocket connection', async () => {
                                          // Test WebSocket connection establishment
                                        });

                                        it('should send messages over WebSocket', async () => {
                                          // Test message sending
                                        });

                                        it('should receive messages over WebSocket', async () => {
                                          // Test message receiving
                                        });

                                        it('should handle WebSocket disconnection', async () => {
                                          // Test disconnection handling
                                        });

                                        it('should reconnect after WebSocket failure', async () => {
                                          // Test reconnection logic
                                        });
                                      });

                                      describe('Control UI Integration', () => {
                                        it('should serve Control UI on correct port', async () => {
                                          // Test port configuration
                                        });

                                        it('should authenticate Control UI requests', async () => {
                                          // Test authentication
                                        });
                                      });

                                      describe('API Endpoints', () => {
                                        it('GET /api/devices should list paired devices', async () => {
                                          // Test device listing endpoint
                                        });

                                        it('POST /api/devices/approve should approve device', async () => {
                                          // Test device approval endpoint
                                        });

                                        it('POST /api/gateway/restart should restart gateway', async () => {
                                          // Test gateway restart endpoint
                                        });
                                      });
                                    });
                                    ```

                                    **Implementation Checklist**:
                                    - [ ] Create integration test file with comprehensive test cases
                                    - [ ] - [ ] Implement fixtures for common test scenarios
                                    - [ ] - [ ] Add test for gateway lifecycle (start, run, stop)
                                    - [ ] - [ ] Add test for device pairing flow
                                    - [ ] - [ ] Add test for WebSocket reliability
                                    - [ ] - [ ] Add test for admin UI access control
                                    - [ ] - [ ] Configure test timeouts for long-running processes
                                    - [ ] - [ ] Add coverage reporting configuration
                                    - [ ] - [ ] Document test running procedures
                                   
                                    - [ ] **Running Tests**:
                                    - [ ] ```bash
                                    - [ ] npm run test                    # Run all tests
                                    - [ ] npm run test:watch            # Run tests in watch mode
                                    - [ ] npm run test:coverage         # Generate coverage report
                                    - [ ] npm run test -- integration   # Run integration tests only
                                    - [ ] ```
                                   
                                    - [ ] ---
                                   
                                    - [ ] ### Task 3: Add Monitoring/Observability
                                   
                                    - [ ] **Objective**: Implement logging, health checks, and operational visibility.
                                   
                                    - [ ] **Files to Create**:
                                    - [ ] - `src/monitoring/logger.ts` - Structured logging
                                    - [ ] - `src/monitoring/health-check.ts` - Health check endpoints
                                    - [ ] - `src/monitoring/metrics.ts` - Metrics collection
                                    - [ ] - `src/routes/health.ts` - Health check routes
                                   
                                    - [ ] **Code Template for `src/monitoring/logger.ts`**:
                                   
                                    - [ ] ```typescript
                                    - [ ] import { MoltbotEnv } from '../types';
                                   
                                    - [ ] export interface LogEntry {
                                    - [ ]   timestamp: string;
                                    - [ ]     level: 'debug' | 'info' | 'warn' | 'error';
                                    - [ ]   service: string;
                                    - [ ]     message: string;
                                    - [ ]   metadata?: Record<string, unknown>;
                                    - [ ]     traceId?: string;
                                    - [ ] }
                                   
                                    - [ ] export class Logger {
                                    - [ ]   constructor(private serviceName: string, private env: MoltbotEnv) {}
                                   
                                    - [ ]     private formatLog(entry: LogEntry): string {
                                    - [ ]     return JSON.stringify(entry);
                                    - [ ]   }
                                   
                                    - [ ]     debug(message: string, metadata?: Record<string, unknown>, traceId?: string) {
                                    - [ ]     const entry: LogEntry = {
                                    - [ ]       timestamp: new Date().toISOString(),
                                    - [ ]         level: 'debug',
                                    - [ ]           service: this.serviceName,
                                    - [ ]             message,
                                    - [ ]               metadata,
                                    - [ ]                 traceId,
                                    - [ ]                 };
                                    - [ ]                 if (this.env.DEBUG_ROUTES) {
                                    - [ ]                   console.log(this.formatLog(entry));
                                    - [ ]                   }
                                    - [ ]                 }
                                   
                                    - [ ]               info(message: string, metadata?: Record<string, unknown>, traceId?: string) {
                                    - [ ]               const entry: LogEntry = {
                                    - [ ]                 timestamp: new Date().toISOString(),
                                    - [ ]                   level: 'info',
                                    - [ ]                     service: this.serviceName,
                                    - [ ]                       message,
                                    - [ ]                         metadata,
                                    - [ ]                           traceId,
                                    - [ ]                           };
                                    - [ ]                           console.log(this.formatLog(entry));
                                    - [ ]                         }
                                   
                                    - [ ]                       warn(message: string, metadata?: Record<string, unknown>, traceId?: string) {
                                    - [ ]                       const entry: LogEntry = {
                                    - [ ]                         timestamp: new Date().toISOString(),
                                    - [ ]                           level: 'warn',
                                    - [ ]                             service: this.serviceName,
                                    - [ ]                               message,
                                    - [ ]                                 metadata,
                                    - [ ]                                   traceId,
                                    - [ ]                                   };
                                    - [ ]                                   console.warn(this.formatLog(entry));
                                    - [ ]                                 }
                                   
                                    - [ ]                               error(message: string, error?: Error, metadata?: Record<string, unknown>, traceId?: string) {
                                    - [ ]                               const entry: LogEntry = {
                                    - [ ]                                 timestamp: new Date().toISOString(),
                                    - [ ]                                   level: 'error',
                                    - [ ]                                     service: this.serviceName,
                                    - [ ]                                       message,
                                    - [ ]                                         metadata: {
                                    - [ ]                                             ...metadata,
                                    - [ ]                                                 error: error?.message,
                                    - [ ]                                                     stack: error?.stack,
                                    - [ ]                                                       },
                                    - [ ]                                                         traceId,
                                    - [ ]                                                         };
                                    - [ ]                                                         console.error(this.formatLog(entry));
                                    - [ ]                                                       }
                                    - [ ]                                                   }
                                    - [ ]                                               ```
                                   
                                    - [ ]                                           **Code Template for `src/monitoring/health-check.ts`**:
                                   
                                    - [ ]                                       ```typescript
                                    - [ ]                                   import { MoltbotEnv } from '../types';
                                   
                                    - [ ]                               export interface HealthStatus {
                                    - [ ]                             status: 'healthy' | 'degraded' | 'unhealthy';
                                    - [ ]                           timestamp: string;
                                    - [ ]                         checks: {
                                    - [ ]                         gateway: 'up' | 'down';
                                    - [ ]                         r2Storage: 'available' | 'unavailable';
                                    - [ ]                         websocket: 'connected' | 'disconnected';
                                    - [ ]                       };
                                    - [ ]                     version: string;
                                    - [ ]                   uptime: number;
                                    - [ ]               }
                                   
                                    - [ ]           export async function checkHealth(env: MoltbotEnv): Promise<HealthStatus> {
                                    - [ ]         const startTime = performance.now();
                                   
                                    - [ ]         // Check gateway status
                                    - [ ]       let gatewayStatus: 'up' | 'down' = 'down';
                                    - [ ]     try {
                                    - [ ]     const processes = await env.Sandbox.listProcesses();
                                    - [ ]     gatewayStatus = processes.some(p => p.name?.includes('moltbot')) ? 'up' : 'down';
                                    - [ ]   } catch (error) {
                                    - [ ]       gatewayStatus = 'down';
                                    - [ ]     }
                                   
                                    - [ ]   // Check R2 storage
                                    - [ ]     let r2Status: 'available' | 'unavailable' = 'unavailable';
                                    - [ ]   if (env.R2_ACCESS_KEY_ID && env.R2_SECRET_ACCESS_KEY) {
                                    - [ ]       r2Status = 'available';
                                    - [ ]     }
                                   
                                    - [ ]   // Check WebSocket (simplified - actual check would connect to gateway)
                                    - [ ]     let wsStatus: 'connected' | 'disconnected' = 'disconnected';
                                    - [ ]   // Implementation depends on WebSocket manager
                                   
                                    - [ ]     const overallStatus = gatewayStatus === 'up' && wsStatus === 'connected'
                                    - [ ]     ? 'healthy'
                                    - [ ]     : gatewayStatus === 'up'
                                    - [ ]     ? 'degraded'
                                    - [ ]     : 'unhealthy';
                                   
                                    - [ ]   return {
                                    - [ ]       status: overallStatus,
                                    - [ ]       timestamp: new Date().toISOString(),
                                    - [ ]       checks: {
                                    - [ ]         gateway: gatewayStatus,
                                    - [ ]           r2Storage: r2Status,
                                    - [ ]             websocket: wsStatus,
                                    - [ ]             },
                                    - [ ]             version: '1.0.0',
                                    - [ ]             uptime: performance.now() - startTime,
                                    - [ ]           };
                                    - [ ]       }
                                    - [ ]   ```
                                   
                                    - [ ]   **Implementation Steps**:
                                    - [ ]   1. Create logger utility with structured logging
                                    - [ ]   2. Create health check system
                                    - [ ]   3. Add health check endpoint at `GET /health`
                                    - [ ]   4. Add metrics collection for key operations
                                    - [ ]   5. Integrate logging throughout codebase
                                    - [ ]   6. Add Grafana dashboard templates (optional)
                                    - [ ]   7. Document observability best practices
                                   
                                    - [ ]   ---
                                   
                                    - [ ]   ### Task 4: Enhance R2 Sync Reliability
                                   
                                    - [ ]   **Objective**: Implement atomic sync operations and data consistency verification.
                                   
                                    - [ ]   **Files to Modify**:
                                    - [ ]   - `src/gateway/r2.ts` - R2 mounting logic
                                    - [ ]   - `src/gateway/sync.ts` - Sync logic
                                   
                                    - [ ]   **Code Template - Add to `src/gateway/r2.ts`**:
                                   
                                    - [ ]   ```typescript
                                    - [ ]   export async function atomicR2Sync(
                                    - [ ]     sandbox: Sandbox,
                                    - [ ]   sourceDir: string,
                                    - [ ]     bucketName: string,
                                    - [ ]   env: MoltbotEnv
                                    - [ ]   ): Promise<{success: boolean; timestamp: string; checksums: Record<string, string>}> {
                                    - [ ]     const logger = new Logger('r2-sync', env);
                                   
                                    - [ ]     try {
                                    - [ ]     // Step 1: Create temporary staging directory
                                    - [ ]     const stagingDir = `${sourceDir}.staging`;
                                    - [ ]     await sandbox.startProcess({
                                    - [ ]       image: 'busybox',
                                    - [ ]         args: ['mkdir', '-p', stagingDir],
                                    - [ ]         });
                                   
                                    - [ ]         // Step 2: Copy files to staging with verification
                                    - [ ]         const checksums: Record<string, string> = {};
                                    - [ ]         await sandbox.startProcess({
                                    - [ ]           image: 'alpine:latest',
                                    - [ ]             args: ['sh', '-c', `find ${sourceDir} -type f -exec sha256sum {} \\; > ${stagingDir}/manifest.sha256`],
                                    - [ ]             });
                                   
                                    - [ ]             // Step 3: Use rsync with proper flags for s3fs
                                    - [ ]             const syncResult = await sandbox.startProcess({
                                    - [ ]               image: 'rsync:latest',
                                    - [ ]                 args: [
                                    - [ ]                     'rsync',
                                    - [ ]                         '-r',
                                    - [ ]                             '--no-times',  // Critical: s3fs doesn't support timestamps
                                    - [ ]                                 '--delete',
                                    - [ ]                                     '--checksum',
                                    - [ ]                                         `${stagingDir}/`,
                                    - [ ]                                             `/data/moltbot/`,
                                    - [ ]                                               ],
                                    - [ ]                                               });
                                   
                                    - [ ]                                               if (syncResult.exitCode !== 0) {
                                    - [ ]                                                 throw new Error(`Rsync failed with exit code ${syncResult.exitCode}`);
                                    - [ ]                                                 }
                                   
                                    - [ ]                                                 // Step 4: Verify sync integrity
                                    - [ ]                                                 const verifyResult = await sandbox.startProcess({
                                    - [ ]                                                   image: 'alpine:latest',
                                    - [ ]                                                     args: ['sh', '-c', `cd /data/moltbot && sha256sum -c ${stagingDir}/manifest.sha256`],
                                    - [ ]                                                     });
                                   
                                    - [ ]                                                     if (verifyResult.exitCode !== 0) {
                                    - [ ]                                                       throw new Error('Data integrity check failed after sync');
                                    - [ ]                                                       }
                                   
                                    - [ ]                                                       // Step 5: Cleanup staging directory
                                    - [ ]                                                       await sandbox.startProcess({
                                    - [ ]                                                         image: 'busybox',
                                    - [ ]                                                           args: ['rm', '-rf', stagingDir],
                                    - [ ]                                                           });
                                   
                                    - [ ]                                                           logger.info('R2 sync completed successfully', { sourceDir, bucketName });
                                   
                                    - [ ]                                                           return {
                                    - [ ]                                                             success: true,
                                    - [ ]                                                               timestamp: new Date().toISOString(),
                                    - [ ]                                                                 checksums,
                                    - [ ]                                                                 };
                                    - [ ]                                                               } catch (error) {
                                    - [ ]                                                               logger.error('R2 sync failed', error as Error, { sourceDir, bucketName });
                                    - [ ]                                                               throw error;
                                    - [ ]                                                             }
                                    - [ ]                                                         }
                                   
                                    - [ ]                                                     export async function verifyR2Mount(sandbox: Sandbox): Promise<boolean> {
                                    - [ ]                                                   try {
                                    - [ ]                                                   const result = await sandbox.startProcess({
                                    - [ ]                                                     image: 'busybox',
                                    - [ ]                                                       args: ['sh', '-c', 'mount | grep s3fs'],
                                    - [ ]                                                       });
                                   
                                    - [ ]                                                           const { stdout } = await result.getLogs();
                                    - [ ]                                                           return stdout.includes('s3fs on /data/moltbot');
                                    - [ ]                                                         } catch (error) {
                                    - [ ]                                                         return false;
                                    - [ ]                                                       }
                                    - [ ]                                                   }
                                    - [ ]                                               ```
                                   
                                    - [ ]                                           **Implementation Steps**:
                                    - [ ]                                       1. Implement atomic sync with staging directory
                                    - [ ]                                   2. Add checksum-based verification
                                    - [ ]                               3. Implement rollback mechanism for failed syncs
                                    - [ ]                           4. Add R2 mount status monitoring
                                    - [ ]                       5. Create sync failure alerts
                                    - [ ]                   6. Document data safety procedures
                                    - [ ]               7. Add backup retention policies
                                   
                                    - [ ]           ---
                                   
                                    - [ ]       ### Task 5: Performance Optimization
                                   
                                    - [ ]   **Objective**: Reduce cold start times and improve throughput.
                                   
                                    - [ ]   **Profiling Steps**:
                                    - [ ]   ```bash
                                    - [ ]   # Measure cold start time
                                    - [ ]   time npm run deploy && time curl https://your-worker.workers.dev/health
                                   
                                    - [ ]   # Generate flame graph
                                    - [ ]   node --prof src/index.ts
                                    - [ ]   node --prof-process isolate-*.log > profile.txt
                                    - [ ]   ```
                                   
                                    - [ ]   **Key Optimizations**:
                                    - [ ]   1. **Container pre-warming**: Keep container alive longer
                                    - [ ]   2. **Lazy loading**: Load skills only when needed
                                    - [ ]   3. **Caching**: Cache compiled routes and configs
                                    - [ ]   4. **Dependency reduction**: Minimize production dependencies
                                    - [ ]   5. **Code splitting**: Split large modules
                                    - [ ]   6. **Compression**: Gzip responses
                                   
                                    - [ ]   **Implementation Checklist**:
                                    - [ ]   - [ ] Profile gateway startup time
                                    - [ ]   - [ ] Optimize container lifecycle settings
                                    - [ ]   - [ ] Implement skill lazy loading
                                    - [ ]   - [ ] Add response compression
                                    - [ ]   - [ ] Cache static assets
                                    - [ ]   - [ ] Reduce bundle size
                                    - [ ]   - [ ] Document performance benchmarks
                                    - [ ]   - [ ] Set up performance monitoring
                                   
                                    - [ ]   ---
                                   
                                    - [ ]   ### Task 6: Additional Chat Platform Support
                                   
                                    - [ ]   **Objective**: Add Teams and WhatsApp integrations following the existing pattern.
                                   
                                    - [ ]   **Template for `src/channels/teams.ts`**:
                                   
                                    - [ ]   ```typescript
                                    - [ ]   import { Logger } from '../monitoring/logger';
                                    - [ ]   import { MoltbotEnv } from '../types';
                                   
                                    - [ ]   export class TeamsChannel {
                                    - [ ]     private logger: Logger;
                                   
                                    - [ ]   constructor(private env: MoltbotEnv) {
                                    - [ ]       this.logger = new Logger('teams-channel', env);
                                    - [ ]     }
                                   
                                    - [ ]   async initialize(): Promise<void> {
                                    - [ ]       if (!this.env.TEAMS_BOT_TOKEN) {
                                    - [ ]         this.logger.warn('Teams bot token not configured');
                                    - [ ]           return;
                                    - [ ]           }
                                    - [ ]           // Initialize Teams bot connection
                                    - [ ]         }
                                   
                                    - [ ]       async sendMessage(userId: string, message: string): Promise<void> {
                                    - [ ]       // Implement message sending to Teams
                                    - [ ]     }
                                   
                                    - [ ]   async handleIncomingMessage(payload: unknown): Promise<void> {
                                    - [ ]       // Implement incoming message handling
                                    - [ ]     }
                                   
                                    - [ ]   async handleDevicePairing(userId: string, deviceId: string): Promise<void> {
                                    - [ ]       // Implement device pairing flow for Teams
                                    - [ ]     }
                                    - [ ] }
                                    - [ ] ```
                                   
                                    - [ ] **Implementation Steps**:
                                    - [ ] 1. Create Teams channel module
                                    - [ ] 2. Create WhatsApp channel module
                                    - [ ] 3. Add environment variables for bot tokens
                                    - [ ] 4. Implement message routing
                                    - [ ] 5. Handle platform-specific features
                                    - [ ] 6. Add error handling and logging
                                    - [ ] 7. Write platform-specific tests
                                    - [ ] 8. Document setup procedures
                                   
                                    - [ ] ---
                                   
                                    - [ ] ### Task 7: Upstream Updates
                                   
                                    - [ ] **Objective**: Track and document OpenClaw/Moltbot upstream changes.
                                   
                                    - [ ] **Create `docs/UPSTREAM_TRACKING.md`**:
                                   
                                    - [ ] ```markdown
                                    - [ ] # Upstream Tracking
                                   
                                    - [ ] ## OpenClaw Project
                                    - [ ] - **Repository**: https://github.com/openclaw/openclaw
                                    - [ ] - **Current Version**: [Version from package.json]
                                    - [ ] - **Last Checked**: [Date]
                                   
                                    - [ ] ## Notable Changes to Track
                                   
                                    - [ ] ### Naming Changes
                                    - [ ] - Tool originally named: `clawdbot`
                                    - [ ] - Renamed to: `openclaw` (in progress)
                                    - [ ] - Internal references still use `clawdbot`
                                    - [ ] - **Status**: Pending upstream merge
                                   
                                    - [ ] ### Package Changes
                                    - [ ] - [@openclaw/cli](https://npmjs.org/@openclaw/cli) - Monitor for updates
                                    - [ ] - [@openclaw/types](https://npmjs.org/@openclaw/types)
                                   
                                    - [ ] ## Migration Checklist
                                    - [ ] - [ ] Update CLI tool references when upstream renames
                                    - [ ] - [ ] Update config path references
                                    - [ ] - [ ] Test compatibility with new versions
                                    - [ ] - [ ] Document breaking changes
                                    - [ ] - [ ] Update AGENTS.md with new patterns
                                    - [ ] - [ ] Test E2E after upstream updates
                                   
                                    - [ ] ## Contribution Plan
                                    - [ ] - Monitor OpenClaw issues and PRs
                                    - [ ] - Contribute moltbot-sandbox insights back to project
                                    - [ ] - Participate in design decisions
                                    - [ ] - Test release candidates
                                    - [ ] ```
                                   
                                    - [ ] **Implementation Steps**:
                                    - [ ] 1. Create upstream tracking documentation
                                    - [ ] 2. Set up dependency monitoring (Dependabot)
                                    - [ ] 3. Create version compatibility matrix
                                    - [ ] 4. Document migration paths
                                    - [ ] 5. Plan major version updates
                                    - [ ] 6. Communicate changes to team
                                   
                                    - [ ] ---
                                   
                                    - [ ] ## Summary
                                   
                                    - [ ] These 7 tasks form a complete roadmap for production-ready OpenClaw deployment on Cloudflare. Implementation should proceed in the order listed, with each task building on previous foundations.
                                   
                                    - [ ] **Estimated Timeline**:
                                    - [ ] - Tasks 1-2: 2-3 weeks (core reliability)
                                    - [ ] - Task 3: 1 week (observability)
                                    - [ ] - Task 4: 1-2 weeks (data safety)
                                    - [ ] - Task 5: 1 week (optimization)
                                    - [ ] - Task 6-7: 1-2 weeks (expansion and maintenance)
                                   
                                    - [ ] **Total**: Approximately 6-9 weeks for full implementation with testing.
                                   
                                    - [ ] Each task includes production-ready code templates and detailed implementation steps that can be executed incrementally.
                                    - [ ] 

- **Process status**: The sandbox API's `proc.status` may not update immediately after a process completes. Instead of checking `proc.status === 'completed'`, verify success by checking for expected output (e.g., timestamp file exists after sync).
