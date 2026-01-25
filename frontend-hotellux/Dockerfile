# Stage 1: Build
FROM node:18-alpine AS build
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install --quiet

# Build the application
COPY . .
RUN npm run build --prod

# Stage 2: Serve with Nginx
FROM nginx:stable-alpine

# Remove default nginx static assets
RUN rm -rf /usr/share/nginx/html/*

# Copy build output to Nginx html folder
# Wildcard used to handle subfolders like dist/project-name/browser
COPY --from=build /app/dist/*/browser/ /usr/share/nginx/html/
# If your Angular version is older (pre-v17), use: 
# COPY --from=build /app/dist/frontend-hotellux/ /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
