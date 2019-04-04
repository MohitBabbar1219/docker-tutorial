FROM node:alpine as builder

WORKDIR /usr/app

COPY package.json .
RUN npm install
COPY . .
RUN npm run build

FROM nginx
# This is used by the deployment service to expose our app
EXPOSE 80
COPY --from=builder /usr/app/build /usr/share/nginx/html
