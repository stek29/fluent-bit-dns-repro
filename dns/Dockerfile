FROM alpine:latest

RUN apk add --no-cache dnsmasq

ENTRYPOINT ["dnsmasq", "--no-daemon", "--log-facility=-"] 
