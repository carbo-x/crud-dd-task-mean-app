# Use Node.js 18 as the base image
FROM node:18-alpine

# Set the working folder inside the container
WORKDIR /app

# Copy package.json first (so Docker caches this layer)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy the rest of the backend code
COPY . .

# Tell Docker this container uses port 8080
EXPOSE 8080

# Command to start the server
CMD ["node", "server.js"]
