
## Proxy

Facilitates reverse proxying via lite express.js server.

Used as a lightweight NGINX alternative when in development mode.

### Example

Note: The rules are in order of precedence.
- Once a request is matched by one of the `match` functions, its destination is decided and no other matching is attempted.

```ts
import { wasRequestProxied, createReverseProxy } from '@cseitz/proxy';
import { WebSocketServer } from 'ws';
import { createServer } from 'http';
import express from 'express';

const port = 5000;

/** Express app */
export const app = express();


/** Node HTTP Server */
export const server = createServer((req, res) => (
    // Only route requests that don't get caught by the proxy
    !proxyRequest(req, res) && app(req, res)
));

/** Websocket Server */
export const wss = new WebSocketServer({
    noServer: true,
})

/** Define reverse proxies (like NGINX) */
const proxyRequest = createReverseProxy({ server }, {
    api: {
        /**
         * This rule prevents requests to the API from being proxied
         * - notice how its missing the `server` block, meaning it acts as a request trap that ensures requests dont get proxied
         * */
        enabled: true,
        match(req, { ws }) {
            if (ws) {
                if (!req.url?.startsWith('/_next'))
                    return true; // Matches non-NextJS websockets
            } else {
                if (req.url?.startsWith('/api'))
                    return true; // Matches API routes
            }
        },
    },
    staff: {
        // This rule proxies requests to the staff portal, which runs one port higher than the API
        enabled: true,
        server: {
            ws: true,
            target: {
                host: 'localhost',
                port: port + 1,
            },
        },
        match(req) {
            // Proxy requests that are on the `staff` subdomain, such as `staff.localhost`
            if (req.headers.host?.includes('staff')) {
                return true;
            }
        },
    },
    web: {
        // This catch-all rule forwards any remaining requests to the website running 2 ports higher.
        enabled: true,
        server: {
            ws: true,
            target: {
                host: 'localhost',
                port: port + 2,
            },
        },
        match(req) {
            // Match any requests
            return true;
        },
    }
})

/** Upgrade websockets */
server.on('upgrade', (req, socket, head) => {
    if (!wasRequestProxied(req)) {
        // The request wasn't proxied, so we can do any websocket logic we want
        wss.handleUpgrade(req, socket, head, ws => {
            wss.emit('connection', ws, req);
        })
    }
});


/** Close connections when the process is about to exit */
process.on('SIGTERM', () => {
    wssHandle.broadcastReconnectNotification();
    server.close();
    wss.close();
});


/** Start the server on the specified port */
server.listen(port, () => {
    console.timeEnd('development proxy ready');
});
```