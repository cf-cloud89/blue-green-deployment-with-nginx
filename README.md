# Blue/Green Deployment with Nginx

This project demonstrates a Blue/Green deployment with auto-failover and manual toggling using Nginx and Docker Compose.

It's designed to run two identical application containers (`blue` and `green`) behind an Nginx reverse proxy. Nginx handles routing 100% of traffic to the primary pool and automatically failing over to the backup pool if the primary becomes unhealthy.

## File Structure

-   `docker-compose.yml`: Orchestrates the `nginx`, `app_blue`, and `app_green` services.
-   `nginx.conf.template`: Nginx config template. Variables (`${...}`) are injected by the nginx-init script.
-   `nginx-init.sh`: The script that runs in the Nginx container to generate the final config from the template.
-   `.env.example`: Provides example environment variables.
-   `README.md`: This file.

## How to Run

1.  **Prepare Environment File:**
    Copy the example `.env` file. You **must** edit this file to add your container image URLs for `BLUE_IMAGE` and `GREEN_IMAGE`.

    ```sh
    cp .env.example .env
    nano .env
    ```

2.  **Make Entrypoint Executable:**
    The `nginx-init.sh` script must have execute permissions to run inside the container.

    ```sh
    chmod +x nginx-init.sh
    ```

3.  **Start Services:**
    Run Docker Compose in detached (`-d`) mode.

    ```sh
    docker-compose up -d
    ```

## How to Test

You can use `curl -v` to see both the response body and the HTTP headers (like `X-App-Pool`).

---

### 1. Test Baseline (Blue Active)

**Setup:** Ensure `ACTIVE_POOL=blue` in your `.env` file and run `docker-compose up -d`.

**Action:** Send a request to the Nginx proxy.

```sh
curl http://localhost:8080/version
```

**Expected Output:** You should see a JSON response, and the headers will include `X-App-Pool: blue.`

---

### 2. Test Auto-Failover (Blue -&gt; Green)

1.  **Induce Chaos on Blue:** Send a `POST` request *directly* to the Blue app's exposed port (`8081`) to tell it to start failing.

    ```sh
    # Tell Blue app to start returning 500 errors
    curl -X POST http://localhost:8081/chaos/start?mode=error
    ```

2.  **Test Nginx Proxy:** Immediately send requests to the main Nginx proxy (`8080`).

    ```sh
    curl http://localhost:8080/version
    ```
**Expected Output:** You should get a `200 OK` response (no error!) and see the header `X-App-Pool: green`. Nginx detected the failure on Blue and automatically retried the request on the Green server, all invisibly to you.

---

### 3. Test Auto-Recovery (Green -&gt; Blue)

1.  **Stop Chaos on Blue:** Tell the Blue app to become healthy again.

    ```sh
    curl -X POST http://localhost:8081/chaos/stop
    ```

2.  **Wait for `fail_timeout`:** Wait about 10-15 seconds. (This allows the `fail_timeout=10s` set in the Nginx config to expire).

3.  **Test Nginx Proxy:** Send another request to the proxy.

    ```sh
    curl http://localhost:8080/version
    ```
**Expected Output:** The response should now come from `X-App-Pool: blue`. Nginx has automatically detected that the primary server is healthy again and has routed traffic back.

---

### 4. Test Manual Toggle (Blue -&gt; Green)

1.  **Stop the services:**
    ```sh
    docker-compose down
    ```

2.  **Edit `.env` file:** Change `ACTIVE_POOL=blue` to `ACTIVE_POOL=green`.

3.  **Start services:**
    ```sh
    docker-compose up -d
    ```

4.  **Test the proxy:**
    ```sh
    curl http://localhost:8080/version
    ```
**Expected Output:** All traffic should now go directly to Green (`X-App-Pool: green`), and Blue (`X-App-Pool: blue`) will now be serving as the backup.
