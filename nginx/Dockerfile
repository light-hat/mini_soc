FROM nginx

RUN rm /etc/nginx/conf.d/default.conf
COPY $CERT_FILE /etc/ssl/.
COPY $KEY_FILE /etc/ssl/.
COPY nginx.conf /etc/nginx/conf.d/