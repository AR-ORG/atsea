FROM node:latest 
WORKDIR /usr/src/atsea/app/react-app
COPY react-app .
EXPOSE 5000
RUN npm install
RUN npm run build
RUN yarn global add serve
CMD ["serve","-s","build"]
