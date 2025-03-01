FROM golang:1.21-alpine
WORKDIR /app
COPY main.go .
RUN go mod init devops-cd-project
RUN go build -o main .
EXPOSE 8080
CMD ["./main"]