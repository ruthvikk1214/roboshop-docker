FROM node:20.20.2-alpine3.23 as builder
RUN mkdir /app
WORKDIR /app
COPY package.json .
COPY server.js .
RUN npm install



FROM node:20.20.2-alpine3.23
WORKDIR /app
EXPOSE 8080
COPY --from=builder /app /app
ENV MONGO="true" \
    MONG0_URL="mongodb://mongodb:27017/catalogue"
CMD ["node", "server.js"]