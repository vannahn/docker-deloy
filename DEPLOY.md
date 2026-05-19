# Hướng dẫn Deploy — docker-vite-test

Dự án **React + TypeScript + Vite**.  
Output build là static files (HTML, JS, CSS) trong thư mục `dist/`.

---

## 1. Build

```bash
npm install
npm run build
```

Kết quả: thư mục `dist/` — có thể deploy lên bất kỳ static server nào.

---

## 2. Deploy với Docker (khuyến nghị cho production)

### Tạo `Dockerfile`

```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve với Nginx
FROM nginx:stable-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Build image & chạy

```bash
docker build -t docker-vite-test .
docker run -d -p 8080:80 docker-vite-test
```

Mở `http://localhost:8080`.

### Docker Compose cho production

Tạo `docker-compose.prod.yml`:

```yaml
services:
  app:
    build: .
    ports:
      - "8080:80"
```

Chạy:

```bash
docker-compose -f docker-compose.prod.yml up -d
```

---

## 3. Deploy lên Vercel (nhanh nhất)

1. Push code lên GitHub/GitLab.
2. Vào [vercel.com](https://vercel.com) → Import repo.
3. Framework: **Vite** (tự động nhận).
4. Deploy — xong.

---

## 4. Deploy lên Netlify

1. Push code lên Git.
2. Vào [netlify.com](https://netlify.com) → Import repo.
3. Build command: `npm run build`
4. Publish directory: `dist`
5. Deploy.

---

## 5. Deploy lên static server (Nginx / Apache / CDN)

Copy nội dung thư mục `dist/` lên server.

### Nginx config mẫu

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/docker-vite-test/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 6. Deploy với Docker Compose (dev hiện tại)

File `docker-compose.yml` hiện tại dành cho **development** (chạy `vite dev --host`):

```bash
docker-compose up -d
```

Truy cập `http://localhost:5173`.

> ⚠️ Không dùng cho production — dùng `vite dev` không tối ưu.

---

## 7. Deploy lên GitHub Pages

### Cấu hình Vite

Vì GitHub Pages chạy ở `https://<user>.github.io/<repo>/`, cần set `base` trong `vite.config.ts`:

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  base: '/<repo>/',   // 👈 thay <repo> bằng tên repo của bạn, VD: '/docker-vite-test/'
})
```

> Nếu dùng **user site** (`<user>.github.io`) thì `base: '/'`.

### Cách 1: GitHub Actions (khuyến nghị)

Tạo file `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dist

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

Sau đó vào GitHub repo → **Settings → Pages** → chọn **Source: GitHub Actions**.

### Cách 2: gh-pages package

```bash
npm install -D gh-pages
```

Thêm script vào `package.json`:

```json
"scripts": {
  "deploy": "npm run build && gh-pages -d dist"
}
```

Chạy:

```bash
npm run deploy
```

### Cách 3: Manual

```bash
npm run build
```

Vào GitHub repo → **Settings → Pages** → **Source**: `Deploy from a branch` → chọn branch `gh-pages` / folder `/ (root)` hoặc dùng action upload artifact.

---

## Biến môi trường

Vite dùng `VITE_` prefix. Khai báo trong `.env`:

```
VITE_API_URL=https://api.example.com
```

Các biến `VITE_*` sẽ được compile vào bundle lúc build.
