# Use a slim Python 3.12.3 image as the base
FROM python:3.12.3-slim

# Set the working directory inside the container
WORKDIR /app

# Copy the requirements file and install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy the application code
COPY app.py .

# Tell Docker that the container listens on port 5000
EXPOSE 5000

# Run the application when the container starts
CMD ["python", "app.py"]
