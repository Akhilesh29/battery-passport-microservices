FROM node:20-alpine

WORKDIR /app

COPY shared ./shared
WORKDIR /app/shared
RUN npm install

COPY data-access-service/package*.json ./data-access-service/
WORKDIR /app/data-access-service
RUN npm install

COPY data-access-service/src ./src

EXPOSE 3002

CMD ["npm", "start"]
