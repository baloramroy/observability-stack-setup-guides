# Prometheus Binary Installation on Linux Using Tarball

## Objective
This SOP outlines the procedure for deploying **Prometheus** on a Linux system using its **binary** distribution. It includes instructions for configuring Prometheus as a **systemd** service, setting up **directories** and **data retention policies**.


## Download the Prometheus tarball (official release)

1. Navigate to the **[Prometheus Downloads page](https://prometheus.io/download/)** and download the latest **Linux AMD64 tarball**. 

    `Or`

2. **Download Binary package using command line:**

    **Syntax:**
    
    - Download using this **link** (replace `X.Y.Z` with the **version** you choose):
    
      ```bash
      sudo wget https://github.com/prometheus/prometheus/releases/download/vX.Y.Z/prometheus-X.Y.Z.linux-amd64.tar.gz
      ```
    
    **Example:**
    
    - If the server has **internet connection** then **download prometheus tarball** using this command
    
      ```bash
      sudo wget https://github.com/prometheus/prometheus/releases/download/v3.7.3/prometheus-3.7.3.linux-amd64.tar.gz
      ```	
      > **Note:** This will download the file to the current working directory
    
    
    - If the server has **no internet**, then use **proxy to download prometheus tarball** using this command
    
      ```bash
      sudo wget \
      -e use_proxy=yes \
      -e http_proxy=http://proxy_ip:port \
      -e https_proxy=http://proxy_ip:port \
      https://github.com/prometheus/prometheus/releases/download/v3.7.3/prometheus-3.7.3.linux-amd64.tar.gz
      ```
      > **Note:** Replace the **proxy_ip:port** with real ip and port
