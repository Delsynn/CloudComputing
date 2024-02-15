# Use the official Nginx image as the base image
FROM nginx:1.25.4

# Permissions
RUN chown -R nginx:nginx /usr/share/nginx/html

# Copy the local index.html file to the Nginx document root
COPY . /usr/share/nginx/html/