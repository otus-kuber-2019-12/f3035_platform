FROM alpine:3.10

COPY http_srv.py /opt/

RUN apk add --no-cache \
    python3 && \
    addgroup appgroup -g 1001 && \
    adduser -SH appuser -u 1001 -G appgroup && \
    mkdir /app && \
    echo "<html><head><title>Homework</title></head><body><h1>It works!</h1></body></html>" > /app/homework.html && \
    chown -R appuser:appgroup /app && \
    chown -R appuser:appgroup /opt/http_srv.py

WORKDIR /app
USER appuser

EXPOSE 8000/tcp

CMD ["python3", "/opt/http_srv.py"]
