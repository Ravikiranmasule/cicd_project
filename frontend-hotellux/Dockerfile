# Stage 1: Build
FROM node:16-alpine AS build
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install --quiet --legacy-peer-deps

# Build the application
COPY . .
RUN npm run build --prod

# Stage 2: Serve with Nginx
FROM nginx:stable-alpine

# Remove default nginx static assets
RUN rm -rf /usr/share/nginx/html/*

# Use the specific project name found in your logs
COPY --from=build /app/dist/luxtavern/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
