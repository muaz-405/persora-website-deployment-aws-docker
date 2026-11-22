# Stage 1 image "static-website": serves the site over HTTP (container port 80 -> host 8080)
# These three steps match the build log in docs/screenshots/p2_02_docker_build_nginx_alpine.jpg
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY . /usr/share/nginx/html
