ARG NODE_BASE_IMAGE=node:20-bookworm-slim
FROM ${NODE_BASE_IMAGE} AS build

WORKDIR /app
COPY package.json package-lock.json ./
ARG NPM_REGISTRY=https://repo.huaweicloud.com/repository/npm/
RUN npm ci --registry="${NPM_REGISTRY}" --replace-registry-host=always --no-audit --no-fund
COPY . .
ENV INKSIGHT_BACKEND_API_BASE=http://backend:8080
RUN npm run build

FROM ${NODE_BASE_IMAGE}
WORKDIR /app
ENV NODE_ENV=production INKSIGHT_BACKEND_API_BASE=http://backend:8080
COPY --from=build /app .
EXPOSE 3000

CMD ["npm", "run", "start"]
