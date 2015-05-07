all: push

TAG = 0.1

mt: mt.go
	CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -ldflags '-w' ./mt.go

image: mt
	sudo docker build -t dalanlan/mounttest:$(TAG) .

push: image
	sudo docker push dalanlan/mounttest:$(TAG)

clean:
	rm -f mt
