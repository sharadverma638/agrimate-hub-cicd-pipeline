# syntax=docker/dockerfile:1

# ---- build ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --no-audit --no-fund
COPY . .
# The app hard-codes http://127.0.0.1:8000 in fetch() calls. Make them relative (/api/...)
# so nginx can proxy to the backend and it works on any host.
RUN grep -rl "http://127.0.0.1:8000" src | xargs -r sed -i "s#http://127.0.0.1:8000##g"
RUN npm run build

# ---- serve (unprivileged nginx, static files only) ----
FROM nginxinc/nginx-unprivileged:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://127.0.0.1:8080/ >/dev/null || exit 1
