# Docker Hub 用户名
DOCKER_USER = yourusername
# 镜像名称
IMAGE_NAME = yourimagename

# 构建 dev 镜像
dev:
	docker build -f Dockerfile -t $(DOCKER_USER)/$(IMAGE_NAME):dev .
	docker push $(DOCKER_USER)/$(IMAGE_NAME):dev

# 构建 test 镜像
test:
	docker build -f Dockerfile -t $(DOCKER_USER)/$(IMAGE_NAME):test .
	docker push $(DOCKER_USER)/$(IMAGE_NAME):test

# 构建 prod 镜像
prod:
	docker build -f Dockerfile -t $(DOCKER_USER)/$(IMAGE_NAME):prod .
	docker push $(DOCKER_USER)/$(IMAGE_NAME):prod
