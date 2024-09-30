FROM python:3.12-slim

# Set the working directory in the container
WORKDIR /app

# `git` is required by python package `supawee`
# `postgresql-dev` and `libffi-dev` are for psycopg[binary,pool] (this is always such a pain to install)
RUN apt-get update && apt-get install -y \
    git \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/

# Install Python dependencies
COPY requirements/slow.txt /app/requirements/slow.txt
RUN pip install --no-cache-dir -r requirements/slow.txt
COPY requirements/common.txt /app/requirements/common.txt
RUN pip install --no-cache-dir -r requirements/common.txt

# Copy everything
COPY . /app

# NOTE: For larger apps gunicorn is suggested (we used that at my previous company)
CMD ["uvicorn", "api.app:app", "--host", "0.0.0.0", "--port", "8080", "--log-level", "info"]
