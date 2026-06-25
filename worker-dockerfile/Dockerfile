FROM n8nio/n8n:1.120.4

USER root
RUN apk add --no-cache chromium

# Install community node + missing dependency
RUN mkdir -p /opt/n8n-custom-nodes && \
    cd /opt/n8n-custom-nodes && \
    npm init -y && \
    npm install n8n-nodes-puppeteer && \
    cd node_modules/n8n-nodes-puppeteer && \
    npm install puppeteer-extra-plugin-user-data-dir

USER node

ENV PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
ENV EXTRA_NODE_MODULES_PATH=/opt/n8n-custom-nodes/node_modules
