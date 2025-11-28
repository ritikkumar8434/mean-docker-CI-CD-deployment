# Stage 1: Build Angular app
FROM node:18-alpine AS build

WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build --prod

# Stage 2: Serve via Nginx
FROM nginx:alpine

# Our Nginx config
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# Built Angular app
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
