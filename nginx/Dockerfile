FROM nginx:1.27-alpine

# Remove the default nginx page
RUN rm -rf /usr/share/nginx/html/*

# Copy our custom content
COPY index.html /usr/share/nginx/html/index.html
COPY nginx.conf /etc/nginx/conf.d/default.conf

# OpenShift runs containers as a random UID — make dirs world-writable
RUN chmod -R g+rwx /var/cache/nginx /var/run /var/log/nginx /usr/share/nginx/html

EXPOSE 8080
