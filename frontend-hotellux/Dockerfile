# Stage 1: Build
FROM node:14.21.3-alpine AS build [cite: 13]
WORKDIR /app 
COPY package*.json ./ 
RUN npm install 
COPY . . 
RUN npm run build --prod [cite: 14]

# Stage 2: Serve
FROM nginx:alpine [cite: 14]
# Ensure 'your-project-name' matches the folder in your /dist directory
COPY --from=build /app/dist/your-project-name /usr/share/nginx/html [cite: 14]
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80 [cite: 14]
CMD ["nginx", "-g", "daemon off;"] [cite: 14]