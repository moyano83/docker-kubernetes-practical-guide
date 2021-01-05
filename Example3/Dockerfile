FROM node:latest

WORKDIR /app

COPY . .

RUN npm install

EXPOSE 80

VOLUME [ "/app/feedback" ]

CMD ["npm", "start"] 