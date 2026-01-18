# Solana Token Deployment Platform 

> NOTE: This repository contains a built/static Next.js export (compiled HTML, CSS and JS) — not the original source files. The README below documents what this export is, how to run it, how to reason about and extend it, and practical examples to rebuild a full development workflow.

Table of contents
- Overview
- What this export contains (file map)
- Quickstart — serve locally
- Inspecting runtime behavior & debugging
- Recreating a development environment (recommended stack)
- Core integration points
  - Wallet connection (example)
  - Token metadata schema (example)
  - Upload flow: IPFS / Arweave example
  - Minting flow with Metaplex (example)
- Extending the Uploader / Metadata UI
- Backend patterns (recommended)
- CI / CD, Docker, and Hosting
- Testing & QA recommendations
- Security checklist
- Troubleshooting
- Appendix: Useful snippets, env vars, and recommended libraries

---

## Overview

This exported build appears to be a frontend scaffold for deploying tokens on Solana. It contains client-side pages for:
- Landing (index)
- Uploader (upload assets / token metadata)
- Metadata preview
- Metadata update

It looks designed to integrate with a Solana wallet (wallet-adapter styling present) and to upload token assets/metadata (uploader/metadata pages). Because the repo includes only the compiled static build, you cannot directly edit React sources here — instead you can:
- Serve the static build for QA / demo
- Recreate the app by starting a Next.js project and wiring the discovered pages/components back into source
- Integrate backend minting/upload endpoints and develop locally

Throughout this README I provide advanced, copy-paste-ready examples for typical development tasks.

---

## What this export contains (file map)

High-level important files:
- `index.html` — main landing page
- `uploader.html` — uploader UI page (upload assets/metadata)
- `metadata.html` — metadata preview page
- `update.html` — update metadata page
- `favicon.ico` — branding icon
- `_next/static/...` — JS/CSS bundles created by Next.js build

Use these mapped filenames when serving or reverse-engineering client flows.

---

## Quickstart — serve the static build locally

Since this repo is a static site, the fastest way to run it is to serve the directory.

Option A — Node static server (http-server):
```
npx http-server -c-1 -p 8080
# Then open http://localhost:8080/index.html
```

Option B — Python 3:
```
python3 -m http.server 8080
# Then open http://localhost:8080/index.html
```

Option C — Docker (simple static server):
```
docker run --rm -it -v "$PWD":/usr/share/nginx/html:ro -p 8080:80 nginx:alpine
# Then open http://localhost:8080/index.html
```

Notes:
- The pages are client-side apps that rely on the JS bundles under `_next/static`.
- Open DevTools (Console / Network) to see runtime interactions, discovered API endpoints, or wallet adapter logs.

---

## Inspecting runtime behavior & debugging

Because this is a built site:
- Inspect the Network tab to see any outbound API calls (uploads, blockchain endpoints, RPC calls).
- Look for calls to known services (Pinata, Infura, Arweave, Metaplex / Candy Machine, or custom endpoints).
- In Console, look for wallet adapter initialization or web3 calls — these often include the package names and functions (e.g., `@solana/wallet-adapter`).
- Use Source -> webpack bundles to view the client code. Search the chunk files for function names and strings like `mint`, `upload`, `metadata`, `arweave`, `ipfs` to find integration points.
- If you need to patch behavior quickly, you can inject a small script on top of the served files to intercept network behavior (for QA only).

---

## Recreating a development environment (recommended stack)

Since the original source is not included, the recommended approach is to recreate a Next.js project and reimplement pages (or extract UI and logic from the built bundles as needed).

Recommended stack:
- Next.js (v12+ or v13 app router if rebuilding)
- React 18+
- TypeScript (recommended)
- @solana/web3.js — Solana RPC client
- @solana/wallet-adapter (wallet connection)
- @metaplex-foundation/js or @metaplex/js — minting & metadata (depending on version)
- IPFS clients: `ipfs-http-client` or Pinata / Web3.Storage SDKs
- TailwindCSS / DaisyUI — UI appears to be Tailwind/DaisyUI-like in build
- ESLint + Prettier

Example package.json (minimal starter):
```json
{
  "name": "solana-token-deploy",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build && next export -o out",
    "start": "next start -p 3000",
    "serve": "serve out -s -l 8080"
  },
  "dependencies": {
    "next": "^13.0.0",
    "react": "^18.0.0",
    "react-dom": "^18.0.0",
    "@solana/web3.js": "^1.73.0",
    "@solana/wallet-adapter-react": "^0.15.0",
    "@solana/wallet-adapter-react-ui": "^0.9.0",
    "@solana/wallet-adapter-wallets": "^0.15.0",
    "@metaplex-foundation/js": "^0.16.0",
    "ipfs-http-client": "^59.0.0"
  }
}
```

Recreate pages:
- Use the exported HTML to copy design and UX.
- Reconstruct components (Navbar, Upload form, Metadata form, Preview).
- Add wallet-adapter wiring at top-level (Provider) to handle wallet connections.

---

## Core integration points (examples)

Below are practical code snippets to integrate wallet, upload metadata, and mint a token.

### 1) Wallet connection (React) — minimal
This example uses `@solana/wallet-adapter-react` and `@solana/wallet-adapter-react-ui`:

```tsx
// app/_app.tsx (or pages/_app.tsx)
import { WalletAdapterNetwork } from '@solana/wallet-adapter-base';
import { ConnectionProvider, WalletProvider } from '@solana/wallet-adapter-react';
import {
  PhantomWalletAdapter,
  SolflareWalletAdapter
} from '@solana/wallet-adapter-wallets';
import { WalletModalProvider } from '@solana/wallet-adapter-react-ui';
import { clusterApiUrl } from '@solana/web3.js';

const network = WalletAdapterNetwork.Devnet;
const endpoint = clusterApiUrl(network);

function MyApp({ Component, pageProps }) {
  const wallets = [new PhantomWalletAdapter(), new SolflareWalletAdapter()];

  return (
    <ConnectionProvider endpoint={endpoint}>
      <WalletProvider wallets={wallets} autoConnect>
        <WalletModalProvider>
          <Component {...pageProps} />
        </WalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
}

export default MyApp;
```

UI:
- Use `<WalletMultiButton />` from `@solana/wallet-adapter-react-ui` for connection.

### 2) Token metadata JSON (SPL/Metaplex standard) — example
This is the standard token metadata format used by Metaplex and many wallets:

```json
{
  "name": "My Token",
  "symbol": "MTK",
  "description": "Token for My Project",
  "seller_fee_basis_points": 500,
  "image": "ipfs://Qm.../image.png",
  "external_url": "https://myproject.example",
  "attributes": [
    { "trait_type": "Level", "value": "1" }
  ],
  "properties": {
    "files": [
      { "uri": "ipfs://Qm.../image.png", "type": "image/png" }
    ],
    "category": "image",
    "creators": [
      { "address": "YourWalletAddress", "share": 100 }
    ]
  }
}
```

### 3) Upload assets to IPFS (pinata / web3.storage example)

Using `ipfs-http-client`:
```js
import { create } from 'ipfs-http-client';

// Example using Infura IPFS (requires project id/secret)
const client = create({
  host: 'ipfs.infura.io',
  port: 5001,
  protocol: 'https',
  headers: {
    authorization: 'Basic ' + Buffer.from(`${INFURA_PROJECT_ID}:${INFURA_PROJECT_SECRET}`).toString('base64')
  }
});

async function uploadFile(file) {
  const added = await client.add(file);
  // example CID
  return `ipfs://${added.path}`;
}
```

Or using `web3.storage`:
```js
import { Web3Storage } from 'web3.storage';
const client = new Web3Storage({ token: process.env.WEB3_STORAGE_TOKEN });

async function uploadFiles(files) {
  const cid = await client.put(files);
  return `ipfs://${cid}/<filename>`;
}
```

### 4) Mint token (Metaplex + @solana/web3.js) — example
This sketch demonstrates a simple metaplex mint for an NFT (single edition). For SPL-Token minting (fungible), you would use `@solana/spl-token`.

```ts
import { keypairIdentity, bundlrStorage, Metaplex } from "@metaplex-foundation/js";
import { Connection, Keypair } from "@solana/web3.js";

async function mintNft(connection: Connection, payerKeypair: Keypair, metadataUri: string) {
  const metaplex = Metaplex.make(connection)
    .use(keypairIdentity(payerKeypair))
    .use(bundlrStorage({address: 'https://node1.bundlr.network', providerUrl: 'https://api.devnet.solana.com', timeout: 60000}));

  const { nft } = await metaplex.nfts().create({
    uri: metadataUri,
    name: "My NFT",
    sellerFeeBasisPoints: 500,
    symbol: "MTK",
    maxSupply: 1
  });

  return nft; // contains mint address and metadata
}
```

Notes:
- Replace with the correct storage plugin (Arweave, Bundlr) according to your integration.
- For SPL tokens (fungible), prefer `@solana/spl-token` and mint tokens via Token program.

---

## Extending the Uploader / Metadata UI

Common extensions for the uploader page:
- Add client-side validation for file types and metadata fields.
- Show a preview of the uploaded image before upload.
- Show progress indicators (upload bytes / total) and final CID/URL.
- Integrate third-party pinning services (Pinata, NFT.Storage, Web3.Storage, Bundlr for Arweave).
- Allow batch uploads and bulk metadata generation (CSV -> JSON).

Architectural advice:
- Keep uploads client-side for decentralization, but also provide a trusted backend (if you need to sign transactions server-side or maintain private keys).
- For large assets, consider uploading to Arweave via Bundlr to ensure permanent storage.

---

## Backend patterns (recommended)

If you add a backend, here are common patterns:

1. Minimal API for signing transactions
- Endpoint: `/api/sign`
- Purpose: sign a serialized transaction with server-side key (for custodial minting).

2. Upload proxy (optional)
- Endpoint: `/api/upload`
- Purpose: receive file(s), store to IPFS/Arweave, return CID(s).
- Use for controlled pinning or when you need to perform additional processing.

3. Metadata registry
- Endpoint: `/api/metadata/:mint`
- Purpose: store and fetch application-level metadata (not necessarily on-chain).

Deployment considerations:
- Rate limit upload endpoints, validate file sizes, use auth for custodial endpoints.
- Use stateless workers (serverless) to scale upload processing; for long uploads use background queue (SQS / RabbitMQ) + CDN.

---

## CI / CD, Docker, and Hosting

Since the app is a static Next.js export:
- Host on Vercel, Netlify, Cloudflare Pages, or any static host.
- In CI, build with `next build && next export` and push `out` artifacts to the hosting provider.

Example GitHub Actions (build + deploy to a static host):
```yaml
name: Build and Export
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18
      - run: npm ci
      - run: npm run build
      - run: npm run export
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: static-site
          path: out
```

Dockerfile (static server)
```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Testing & QA recommendations

- Unit tests: use Jest + React Testing Library for components.
- Integration tests: Playwright or Cypress for end-to-end flows (connect wallet via mocks or use Phantom headless).
- Use a local Solana test validator for integration tests:
  ```
  solana-test-validator --reset
  ```
- For minting tests, use local wallets with known keypairs and local RPC (or Devnet).

---

## Security checklist

- Never commit private keys or secrets. Use environment variables or secret stores.
- For custodial signing endpoints, require authentication and restrict by IP where possible.
- Sanitize and limit file uploads by type and size.
- Rate limit endpoints to avoid abusive usage and cost spikes (e.g., Bundlr/Arweave).
- Use HTTPS in production and secure CORS on API endpoints.

---

## Troubleshooting

- Page loads but wallet doesn't appear:
  - Confirm the wallet extension (e.g., Phantom) is installed.
  - Open DevTools and search for wallet adapter logs.
- Upload fails:
  - Check network requests to see which endpoint is being called.
  - Ensure CORS is configured for third-party pinning services.
- Mint fails with RPC error:
  - Validate network (Devnet vs Mainnet).
  - Check payer wallet balance for SOL to pay transaction fees.
  - Inspect transaction logs (simulate or run `solana logs` on validator).

---

## Appendix

Recommended libraries
- @solana/web3.js — Solana RPC client
- @solana/spl-token — SPL Token utilities
- @solana/wallet-adapter/* — Wallet connection components
- @metaplex-foundation/js — Metaplex NFT and metadata utilities
- ipfs-http-client / web3.storage / nft.storage / bundlr-client — storage providers

Sample environment variables
```
RPC_URL=https://api.devnet.solana.com
INFURA_PROJECT_ID=...
INFURA_PROJECT_SECRET=...
WEB3_STORAGE_TOKEN=...
BUNDLR_URL=https://node1.bundlr.network
```

Useful commands
- Serve the static export: `npx http-server -c-1 -p 8080`
- Rebuild from Next.js source: `next build && next export -o out`

File map (where to look in the export)
- index.html -> main UI (home/landing)
- uploader.html -> upload flow
- metadata.html -> metadata preview page
- update.html -> metadata update flow
- _next/static/css/* -> styles (Tailwind-like)
- _next/static/chunks/pages/* -> compiled page logic (searchable for function names and strings)

---
