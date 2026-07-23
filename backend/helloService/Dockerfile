FROM node:18-alpine
WORKDIR /app
COPY package*json ./
RUN npm install
COPY . .
# Create non-root user (Alpine uses adduser, not useradd)
RUN adduser -D -s /bin/sh appuser

# Give ownership of both /app and /home/appuser to appuser
RUN chown -R appuser:appuser /app /home/appuser

# Switch to the new user
USER appuser

EXPOSE 3001
CMD ["node", "index.js"]