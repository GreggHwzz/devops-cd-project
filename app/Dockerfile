# Utiliser l'image officielle Go
FROM golang:1.21-alpine

# Définir le répertoire de travail
WORKDIR /app

# Copier le fichier main.go
COPY main.go .

# Initialiser le module Go
RUN go mod init devops-cd-project

# Compiler l'application
RUN go build -o main .

# Exposer le port 8080
EXPOSE 8080

# Commande pour démarrer l'application
CMD ["./main"]