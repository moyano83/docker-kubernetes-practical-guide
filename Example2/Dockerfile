FROM node:latest

RUN mkdir /app

WORKDIR /app

# The first dot is the folder to copy from (same than where the Dockerfile is)
# The second dot is the destination (in this case the workdir)
COPY . .

RUN npm install

# We need to expose the port that node listens to with -p <LOCAL_PORT>:80
EXPOSE 80

CMD ["node", "server.js"]