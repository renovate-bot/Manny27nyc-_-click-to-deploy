include ../crd.Makefile
include ../gcloud.Makefile
include ../var.Makefile
include ../images.Makefile

CHART_NAME := solr
APP_ID ?= $(CHART_NAME)
WAIT_FOR_READY_TIMEOUT := 600

TRACK ?= 9.6
ZK_TAG ?= 3.8
SERVICE_NAME := services/effa00b5-9dfc-4626-aa25-85435f332575.cloudpartnerservices.goog

METRICS_EXPORTER_TAG ?= v0.11.1-gke.1

SOURCE_REGISTRY ?= marketplace.gcr.io/google
IMAGE_SOLR ?= $(SOURCE_REGISTRY)/solr9:$(TRACK)
IMAGE_PROMETHEUS_TO_SD ?= gke.gcr.io/prometheus-to-sd:$(METRICS_EXPORTER_TAG)
IMAGE_ZOOKEEPER ?= $(SOURCE_REGISTRY)/zookeeper3:$(ZK_TAG)

# Main image
image-$(CHART_NAME) := $(call get_sha256,$(IMAGE_SOLR))

# List of images used in application
ADDITIONAL_IMAGES := zookeeper prometheus-to-sd

# Additional images variable names should correspond with ADDITIONAL_IMAGES list
image-zookeeper := $(call get_sha256,$(IMAGE_ZOOKEEPER))
image-prometheus-to-sd := $(call get_sha256,$(IMAGE_PROMETHEUS_TO_SD))

C2D_CONTAINER_RELEASE := $(call get_c2d_release,$(image-$(CHART_NAME)))

BUILD_ID := $(shell date --utc +%Y%m%d-%H%M%S)
RELEASE ?= $(C2D_CONTAINER_RELEASE)-$(BUILD_ID)
NAME ?= $(APP_ID)-1

# Additional variables
ifdef METRICS_EXPORTER_ENABLED
  METRICS_EXPORTER_ENABLED_FIELD = , "metrics.exporter.enabled": $(METRICS_EXPORTER_ENABLED)
endif

ifdef STANDALONE_MODE_ENABLED
  STANDALONE_MODE_ENABLED_FIELD = , "solr.standalone": $(STANDALONE_MODE_ENABLED)
endif

APP_PARAMETERS ?= { \
  "name": "$(NAME)", \
  "namespace": "$(NAMESPACE)" \
  $(METRICS_EXPORTER_ENABLED_FIELD) \
  $(STANDALONE_MODE_ENABLED_FIELD) \
}

# c2d_deployer.Makefile provides the main targets for installing the application.
# It requires several APP_* variables defined above, and thus must be included after.
include ../c2d_deployer.Makefile


# Build tester image
app/build:: .build/$(CHART_NAME)/tester
