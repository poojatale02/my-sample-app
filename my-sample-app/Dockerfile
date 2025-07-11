# Use an official Nginx base image from Docker Hub.
# This provides a stable and secure foundation for your web server.
FROM nginx:latest

# Remove the default Nginx configuration file.
# This is done to prevent conflicts with our custom configuration.
RUN rm /etc/nginx/conf.d/default.conf

# Copy your custom index.html file into the Nginx web root directory inside the container.
# This is the webpage that Nginx will serve.
COPY index.html /usr/share/nginx/html/index.html

# Copy your custom Nginx configuration file into the appropriate directory inside the container.
# This configuration tells Nginx how to serve your index.html.
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80. This informs Docker that the container listens on the specified network ports
# at runtime. It's a documentation step and doesn't actually publish the port.
EXPOSE 80

# Define the command to run when the container starts.
# "nginx -g 'daemon off;'" keeps Nginx running in the foreground,
# which is necessary for Docker containers.
CMD ["nginx", "-g", "daemon off;"]
