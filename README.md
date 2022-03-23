# Alignment appyter

The alignment appyter uses the xalign package to align fastq files on the CAVATICA platform.

## Execute Appyter locally

Using the Appyter profile for correct visualization
```
python3 -m appyter --profile=biojupies xalign.ipynb
```

### Running Appyter with CAVATICA Resources
```bash
APPYTER_NAME=$(jq -r '.name | ascii_downcase' appyter.json)
APPYTER_VERSION=$(jq -r '.version' appyter.json)
APPYTER_IPYNB=$(jq -r '.appyter.file' appyter.json)
LIBRARY_VERSION=$(appyter --version | awk '{print $3}')
DOCKER_TAG=maayanlab/appyter-${APPYTER_NAME}:${APPYTER_VERSION}-${LIBRARY_VERSION}
```

#### Preparation
```bash
appyter dockerize ${APPYTER_IPYNB} > Dockerfile
appyter nbinspect cwl -i appyter.json ${APPYTER_IPYNB} | jq '.' > ${APPYTER_NAME}.cwl
docker build --build-arg "appyter_version=appyter[production]@git+https://github.com/Maayanlab/appyter@v${LIBRARY_VERSION}" -t $DOCKER_TAG .
docker push $DOCKER_TAG
```

#### Running
Assuming the image is processed and on dockerhub, the below instructions let you launch appyters which do all applicable operations using CAVATICA's file storage (SBFS) & CAVATICA's workflow execution service (WES).

```bash
# TODO: get this variable from https://cavatica.sbgenomics.com/developer/token
export CAVATICA_API_KEY=
# TODO: this is `owner_username/project_name` (as seen in the url of a CAVATICA project)
export CAVATICA_PROJECT=
# TODO: this is the public url your appyter is being hosted behind a TLS terminated load balancer (required for realtime status)
export APPYTER_PUBLIC_URL=

# SBFS config
export APPYTER_DATA_DIR="writecache::chroot::sbfs://${CAVATICA_PROJECT}/#?sbfs.api_endpoint=https://cavatica-api.sbgenomics.com&sbfs.auth_token=${CAVATICA_API_KEY}"
# SBFS config
export APPYTER_DISPATCHER="cavatica?cwl=${APPYTER_NAME}.cwl&project=${CAVATICA_PROJECT}&auth_token=${CAVATICA_API_KEY}"
export APPYTER_PROXY=true
export APPYTER_DEBUG=false
export APPYTER_PROFILE=$(jq -rc '.appyter.profile' appyter.json)
export APPYTER_EXTRAS=$(jq -rc '.appyter.extras' appyter.json)

# actually run the appyter
docker run \
  -e APPYTER_DATA_DIR \
  -e APPYTER_DISPATCHER \
  -e APPYTER_PROFILE \
  -e APPYTER_EXTRAS \
  -e APPYTER_IPYNB \
  -p 5000:5000 \
  -it ${DOCKER_TAG}
```
