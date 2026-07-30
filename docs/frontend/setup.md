# Shieldgrid Web — Local Development Setup

## Prerequisites
- Node.js 20+
- `npm` 10+
- Running instance of [`shieldgrid-core`](https://github.com/Shieldgrid/shieldgrid-core) listening on `http://localhost:3000`

## Installation & Running

1. Clone the repository:
   ```bash
   git clone https://github.com/Shieldgrid/shieldgrid-web.git
   cd shieldgrid-web
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Configure environment variables:
   ```bash
   cp .env.example .env
   ```
   Ensure `VITE_API_BASE_URL` is set to your `shieldgrid-core` endpoint (default: `http://localhost:3000`).

4. Start the development server:
   ```bash
   npm run dev
   ```
   Access the dashboard at `http://localhost:5173`.

5. Run local CI checks before pushing:
   ```bash
   ./scripts/ci-local.sh
   ```
