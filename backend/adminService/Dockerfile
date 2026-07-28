FROM node:18-alpine AS base

WORKDIR /app/adminService

COPY /package*.json ./
RUN npm install --production

COPY /. ./
ENV NODE_ENV=production
EXPOSE 3003

CMD ["npm", "run", "start"]
