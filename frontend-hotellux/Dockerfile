# Stage 1: Build
# Use Node 16 to match your project's specific requirements
FROM node:16-alpine AS build
WORKDIR /app

# Install dependencies
COPY package*.json ./
# Using --legacy-peer-deps helps avoid version conflicts with older Angular projects
RUN npm install --quiet --legacy-peer-deps

# Build the application
COPY . .
RUN npm run build --prod

# Stage 2: Serve with Nginx
FROM nginx:stable-alpine

# Remove default nginx static assets
RUN rm -rf /usr/share/nginx/html/*

# Copy build output to Nginx html folder
# For Angular 16 (your version), the path is usually just dist/project-name
COPY --from=build /app/dist/*/ /usr/share/nginx/html/
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
