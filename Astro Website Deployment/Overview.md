# Astro CI/CD with Jenkins & Nginx (Atomic Deployment)

## **1. Architecture Overview**

We are building a **Zero-Downtime Atomic Deployment** pipeline.

- **Server A (Jenkins):** Builds the application using `pnpm`.

- **Server B (Web Server):** Serves the application using Nginx.

- **Strategy:** Jenkins uploads the build to a timestamped folder on Server B, then updates a symbolic link (`current`) to point to the new build.

---

## **2. Prerequisites**

- **Server A:** Jenkins installed and running (accessible on port 8080).
    
- **Server B:** Linux Server (RHEL/CentOS/AlmaLinux assumed based on SELinux usage).
    
- **GitHub Repository:** Contains your Astro project.

