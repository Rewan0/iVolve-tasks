
FROM node:22-alpine AS build

WORKDIR /app

COPY package.json ./

RUN npm install --omit=dev


FROM node:22-alpine

WORKDIR /app

COPY --from=build /app/node_modules ./node_modules

COPY package.json app.js ./

RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx

EXPOSE 3000

CMD ["node", "app.js"]
