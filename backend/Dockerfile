FROM node:20-alpine
# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
# Switch to non-root user
USER appuser
EXPOSE 5000
CMD ["node", "index.js"]
