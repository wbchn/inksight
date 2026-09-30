ARG PYTHON_BASE_IMAGE=python:3.11-slim-bookworm
FROM ${PYTHON_BASE_IMAGE}

WORKDIR /app

RUN sed -i 's#http://deb.debian.org#https://repo.huaweicloud.com#g' /etc/apt/sources.list.d/debian.sources \
    && apt-get update \
    && apt-get install -y --no-install-recommends g++ \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
ARG PIP_INDEX_URL=https://repo.huaweicloud.com/repository/pypi/simple
RUN pip install --no-cache-dir --index-url "${PIP_INDEX_URL}" -r requirements.txt

COPY . .
RUN python scripts/setup_fonts.py \
    && python scripts/build_native_dither.py

ENV PYTHONUNBUFFERED=1 INKSIGHT_DATA_DIR=/data
EXPOSE 8080

CMD ["python", "-m", "uvicorn", "api.index:app", "--host", "0.0.0.0", "--port", "8080"]
