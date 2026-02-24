# ---- Stage 1: Build the Angular app ----
FROM node:18-alpine AS builder

# Set working folder
WORKDIR /app

# Copy package.json and install dependencies
COPY package*.json ./
RUN npm install

# Fix: pin @types/node to match Angular 15 / TypeScript 4.8
RUN npm install @types/node@18.11.18 --save-dev

# Copy all frontend code
COPY . .

# Build the Angular app for production
RUN npm run build -- --configuration production


# ---- Stage 2: Serve the built app with Nginx ----
FROM nginx:alpine

# Remove the default Nginx welcome page
RUN rm -rf /usr/share/nginx/html/*

# Copy the built Angular files from Stage 1
COPY --from=builder /app/dist/angular-15-crud /usr/share/nginx/html

# Copy our custom Nginx config (handles Angular routing)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]
