# The project is formed following the good practices for structuring projects in Golang specified in https://github.com/golang-standards/project-layout
cmd: In this folder are the input files to the project (main.go) as well as other files depending on the type of execution (eg database.db, config.yml, etc)
/docs: In this folder the swagger configuration files are saved to automatically generate the API documentation (available at http://localhost:3000/docs/index.html)
/internal: The application code is divided into two folders: api and pkg:
/internal/api: Main application, controllers, middlewares and router
/internal/pkg: Models, persistence and database.
/pkg: Modules reusable in other applications (For example: crypto, handlers, etc)
/test: Project unit tests
Makefile: It facilitates the use of the application, it has the following format

get-docs:
	go get -u github.com/swaggo/swag/cmd/swag

docs: get-docs
	swag init --dir cmd/api --parseDependency --output docs

build:
	go build -o bin/restapi cmd/api/main.go

run:
	go run cmd/api/main.go

test:
	go test -v ./test/...

build-docker: build
	docker build . -t api-rest

run-docker: build-docker
	docker run -p 3000:3000 api-rest
  
  
# ********************** alternate dockerfile ************************
FROM golang:1.12-alpine

RUN apk add --no-cache git gcc g++

# Set the Current Working Directory inside the container
WORKDIR /app

# We want to populate the module cache based on the go.{mod,sum} files.
COPY go.mod .
COPY go.sum .

RUN go mod download

COPY . .

# Build the Go app
RUN go build -o ./out/app ./cmd/api/main.go


# This container exposes port 8080 to the outside world
EXPOSE 3000

# Run the binary program produced by `go install`
CMD ["./out/app"]

#################################################################

FROM golang:1.12-alpine AS build_base

RUN apk add --no-cache git  git gcc g++

# Set the Current Working Directory inside the container
WORKDIR /tmp

# We want to populate the module cache based on the go.{mod,sum} files.
COPY go.mod .
COPY go.sum .

RUN go mod download

COPY . .

# Build the Go app
RUN go build -o ./out/app ./cmd/api/main.go

# Start fresh from a smaller image
FROM alpine:3.9
RUN apk add ca-certificates

COPY --from=build_base /tmp/out/app /app/restapi

# This container exposes port 8080 to the outside world
EXPOSE 3000

# Run the binary program produced by `go install`
CMD ["/app/restapi"]
