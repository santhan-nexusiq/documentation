# 12-12-25--image proctoring docker image is failing: root cause analysis:- 
## reasons:
### reason1:
1.our build is multi-architecture --linux/amd64, linux/arm64
2.The amd64 build is working, but the arm64 build failing at:
	ERROR: No matching distribution found for mediapipe==0.10.21
3.This means:mediapipe==0.10.21 does NOT exist for:
	Python 3.10 -- Your base image is using Python 3.10
	Linux ARM64 (aarch64)
4.Mediapipe 0.10.21 requires: Python ≥ 3.11 OR only wheels for amd64, not arm64
5.So the build breaks only for ARM64.

### reason2: 
1.mediapipe 0.10.21 has NO ARM64 wheels
2.Available mediapipe versions for ARM64 are only:
0.10.5, 0.10.7, 0.10.8, 0.10.9, ...0.10.18 (latest supported for ARM64), But NOT 0.10.21.

### reason3: 
1.Warning: Undefined variable $PYTHONPATH
2.Dockerfile includes something like:
	ENV PYTHONPATH=$PYTHONPATH:/app/src
3.But $PYTHONPATH is empty, causing:
	UndefinedVar: Usage of undefined variable '$PYTHONPATH'
4.Not fatal, but an issue.

developer fixes:
---------------
1.Inform the team that mediapipe==0.10.21 is not compatible with ARM64, dev team to change mediapipe version:
mediapipe==0.10.18
This works on:
✔ Python 3.10
✔ ARM64
✔ AMD64 
2. Remove mediapipe from common requirements --> If used only in amd64 ML pipeline	Dev team
	Check whether mediapipe is required at runtime on arm64 (maybe some ML-only microservice only runs on amd64). If the code does not need mediapipe on arm64, remove it from common requirements.txt and put it in requirements-ml.txt.
3. Provide separate requirement files per architecture like requirements-amd64.txt (contains mediapipe==0.10.21)
requirements-arm64.txt (contains mediapipe==0.10.18 OR no mediapipe if not needed)

devops fixes: 
------------
1. Add conditional install in Dockerfile using ARG TARGETARCH

	# at top of Dockerfile, where ARGs are defined
	ARG TARGETARCH
	# avoid UndefinedVar for PYTHONPATH by setting explicit value
	ENV PYTHONPATH=/app/src

	# copy both arch-specific requirements files into the image
	COPY requirements.amd64.txt requirements.arm64.txt ./

	# Install Python deps (uses BuildKit cache if available)
	RUN --mount=type=cache,target=/root/.cache/pip \
		python -m pip install --upgrade pip setuptools wheel \
		&& if [ -f requirements.${TARGETARCH}.txt ]; then \
			 python -m pip install --no-cache-dir -r requirements.${TARGETARCH}.txt; \
		   else \
			 echo "No requirements.${TARGETARCH}.txt found; fallback to requirements.amd64.txt" && \
			 python -m pip install --no-cache-dir -r requirements.amd64.txt; \
		   fi

2. Fix undefined $PYTHONPATH warning (your Dockerfile had ENV PYTHONPATH=$PYTHONPATH:/app/src):
ENV PYTHONPATH=/app/src

3.Create per-arch requirements files (fast options)
A) Remove mediapipe from arm64 requirements (if the app can run without mediapipe on arm64)
	cp requirements.txt requirements.amd64.txt
	# create arm64 file without mediapipe
	grep -v '^mediapipe' requirements.txt > requirements.arm64.txt
B)Pin mediapipe to an arm64-compatible version (example: 0.10.18)
	cp requirements.txt requirements.amd64.txt
	cp requirements.txt requirements.arm64.txt
	# replace any mediapipe line with a pinned compatible version
	sed -i 's/^mediapipe==.*/mediapipe==0.10.18/' requirements.arm64.txt || true
	# if sed didn't match (no mediapipe line), append it explicitly:
	grep -q '^mediapipe' requirements.arm64.txt || echo 'mediapipe==0.10.18' >> requirements.arm64.txt
Pick A or B depending on dev team acceptance

4. Build command to run (exact)
1.Use --push to publish the multi-arch manifest to ECR
If you’re building from repo root locally (recommended — clone first), run:
# from repo directory (safer: clone repo locally first)
	docker buildx build \
	  --platform linux/amd64,linux/arm64 \
	  --no-cache \
	  --build-arg COMMIT_SHA="$(git rev-parse --verify HEAD)" \
	  --build-arg REPO_URL="https://github.com/mahesh/nexus-image-proctoringgg.git" \
	  --build-arg REPO_BRANCH="devops-dev" \
	  --label org.opencontainers.image.revision="$(git rev-parse --verify HEAD)" \
	  -t 76739790000.dkr.ecr.ap-south-2.amazonaws.com/imageproctoring:$(git rev-parse --verify HEAD) \
	  --push \
	  .
note: 
You were previously embedding github_token in REPO_URL. Don’t pass secrets as plain --build-arg (risk: process list leakage, image metadata). Safer options are: 
Clone repo locally and run docker buildx build from repo context. (Best and simplest) 