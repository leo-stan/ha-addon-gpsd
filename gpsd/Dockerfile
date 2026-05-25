ARG BUILD_FROM
FROM ${BUILD_FROM}

# Install gpsd
RUN apk add --no-cache gpsd

# Copy entrypoint
COPY run.sh /
RUN chmod a+x /run.sh

CMD [ "/run.sh" ]
