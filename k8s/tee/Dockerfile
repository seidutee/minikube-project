FROM nginx:alpine

# Clean default nginx files
RUN rm -rf /usr/share/nginx/html/*

# Copy site files (HTML + image)
COPY index.html /usr/share/nginx/html/
COPY fashion.jpeg /usr/share/nginx/html/

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]