# syntax=docker/dockerfile:1
FROM alpine:3.20
ARG BUST=0
RUN --mount=type=cache,target=/cache,id=demo \
    [ -f /cache/marker ] && echo WARM || echo COLD; \
    date +%s > /cache/marker