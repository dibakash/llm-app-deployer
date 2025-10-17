---
title: Llm App Deployer
emoji: 📉
colorFrom: red
colorTo: gray
sdk: docker
pinned: false
license: mit
short_description: 'Project: LLM Code Deployment'
---

Check out the configuration reference at https://huggingface.co/docs/hub/spaces-config-reference

API_ENDPOINT: https://dibakash-llm-app-deployer.hf.space/ready


# LLM App Deployer

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)

An automated web service that receives task requests, generates complete web applications using Gemini AI, deploys them to GitHub Pages, and notifies evaluation endpoints—all without manual intervention.

**API Endpoint:** `https://dibakash-llm-app-deployer.hf.space/ready`

## 🎯 Overview

This service is designed for the **LLM Code Deployment** project, where applications are built, deployed, evaluated, and iteratively improved through automated workflows. It handles multi-round development cycles, supports file attachments, and maintains surgical precision during code updates.

### Key Features

- **Automated Code Generation**: Leverages Gemini 2.5 Flash to generate complete, responsive single-file HTML applications
- **GitHub Integration**: Automatically creates repositories, commits code, and configures GitHub Pages
- **Round-based Updates**: Supports iterative development with surgical code modifications that preserve existing functionality
- **Attachment Handling**: Processes and integrates image attachments (data URIs and URLs) into generated applications
- **Robust Error Handling**: Implements exponential backoff, retry logic, and safe-mode protections
- **Evaluation Integration**: Automatically notifies evaluation servers with deployment metadata

## 📋 Requirements

- Python 3.11+
- FastAPI
- Git
- GitHub Personal Access Token with repo and pages permissions
- Google Gemini API Key

## 🚀 Quick Start

### Installation

1. Clone the repository:
```bash
git clone https://github.com/dibakash/llm-app-deployer.git
cd llm-app-deployer
```

2. Install dependencies:
```bash
pip install -r requirements.txt
```

3. Configure environment variables in `.env`:
```env
GEMINI_API_KEY=your_gemini_api_key_here
GITHUB_TOKEN=your_github_token_here
GITHUB_USERNAME=your_github_username
STUDENT_SECRET=your_secret_key_here
LOG_FILE_PATH=logs/app.log
MAX_CONCURRENT_TASKS=2
KEEP_ALIVE_INTERVAL_SECONDS=30
```

4. Run the service:
```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

### Docker Deployment

```bash
docker build -t llm-app-deployer .
docker run -p 8000:8000 --env-file .env llm-app-deployer
```

## 📡 API Reference

### POST `/ready`

Accepts task requests and initiates the build-deploy-notify workflow.

**Request Body:**
```json
{
  "task": "captcha-solver-abc123",
  "email": "student@example.com",
  "round": 1,
  "brief": "Create a captcha solver that handles ?url=https://.../image.png",
  "evaluation_url": "https://example.com/notify",
  "nonce": "ab12-cd34-ef56",
  "secret": "your_secret_key",
  "attachments": [
    {
      "name": "sample.png",
      "url": "data:image/png;base64,iVBORw..."
    }
  ]
}
```

**Response:**
```json
{
  "status": "ready",
  "message": "Task captcha-solver-abc123 received and processing started."
}
```

### GET `/status`

Returns current service status and last received task information.

**Response:**
```json
{
  "last_received_task": {
    "task": "captcha-solver-abc123",
    "email": "student@example.com",
    "round": 1,
    "brief": "Create a captcha solver...",
    "time": "2025-10-18T12:34:56.789Z"
  },
  "running_background_tasks": 1
}
```

### GET `/health`

Health check endpoint.

### GET `/logs?lines=200`

Retrieves the most recent log entries (default: 200 lines, max: 5000).

## 🔧 How It Works

### Round 1: Initial Generation

1. **Request Validation**: Verifies the student secret matches configuration
2. **Repository Setup**: Creates a new GitHub repository with the task name
3. **Code Generation**: Sends the brief and attachments to Gemini AI with structured prompts
4. **File Creation**: Generates `index.html`, `README.md`, and `LICENSE` (MIT)
5. **Attachment Integration**: Saves attachment files and references them correctly in HTML
6. **Deployment**: Commits all files, pushes to GitHub, and configures Pages
7. **Notification**: POSTs deployment details to the evaluation URL

### Round 2+: Surgical Updates

1. **Context Loading**: Reads existing `index.html` from the repository
2. **Minimal Changes**: Uses a specialized prompt that emphasizes preserving core functionality
3. **Safety Checks**: Validates that LLM output isn't destructively different (length checks, content validation)
4. **Selective Updates**: Only modifies files that need changes, preserves others
5. **Redeployment**: Commits updated code and notifies evaluation server

### Attachment Handling

The service intelligently processes attachments:
- **Image Data URIs**: Extracts base64 content for Gemini's multimodal processing
- **External URLs**: Fetches and encodes images for LLM context
- **File References**: Explicitly lists attachment filenames in prompts to ensure correct HTML references (e.g., `<img src="sample.png">`)
- **Local Storage**: Saves all attachments in the repository directory for GitHub Pages deployment

## 🛡️ Safety Features

### Safe Mode (Round 2+)

- **Length Validation**: Rejects LLM outputs that are suspiciously shorter than original code (less than 30% or 200 characters)
- **Fallback Protection**: Reverts to existing code if LLM returns empty or invalid responses
- **Preserve Core Files**: Maintains `README.md` and `LICENSE` if LLM doesn't modify them

### Error Handling

- **Exponential Backoff**: Retries failed API calls with 1s, 2s, 4s, 8s delays
- **GitHub Pages Timing**: Handles race conditions during Pages configuration with intelligent retries
- **Graceful Degradation**: Continues processing even if individual attachments fail
- **Comprehensive Logging**: Tracks all operations with timestamps and context

## 🧩 Code Architecture

```
main.py
├── Settings & Configuration (Pydantic settings with .env support)
├── Logging Infrastructure (Console + file handlers with flush mechanisms)
├── Data Models (TaskRequest, Attachment using Pydantic)
├── Utility Functions
│   ├── verify_secret() - Authentication
│   ├── safe_makedirs() - Directory management
│   └── remove_local_path() - Cleanup with permission handling
├── Attachment Processing
│   ├── is_image_data_uri() - Data URI validation
│   ├── data_uri_to_gemini_part() - Base64 extraction
│   └── attachment_to_gemini_part() - URL fetching & encoding
├── File Management
│   ├── save_generated_files_locally() - Write LLM output
│   └── save_attachments_locally() - Process & save attachments
├── GitHub Operations
│   ├── setup_local_repo() - Initialize or clone repository
│   └── commit_and_publish() - Git operations & Pages configuration
├── LLM Integration
│   ├── call_gemini_api() - Gemini API with retry logic
│   └── call_llm_round2_surgical_update() - Specialized round 2 prompt
├── Notification
│   └── notify_evaluation_server() - POST deployment metadata
├── Main Workflow
│   └── generate_files_and_deploy() - Orchestrates entire pipeline
└── FastAPI Endpoints
    ├── POST /ready - Task submission
    ├── GET /status - Service status
    ├── GET /health - Health check
    └── GET /logs - Log retrieval
```

## 🎓 Educational Context

This project is part of a larger educational framework where:

1. **Students** build automated deployment systems (this service)
2. **Instructors** send task requests via POST API
3. **Systems** generate, deploy, and evaluate applications
4. **Rounds** iterate on the same codebase with new requirements

The service emphasizes:
- Real-world API integration patterns
- LLM prompt engineering for code generation
- Git workflow automation
- Error handling and reliability
- Security considerations (secret validation, no credentials in commits)

## 📊 Configuration Options

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `GEMINI_API_KEY` | Required | Google Gemini API authentication |
| `GITHUB_TOKEN` | Required | GitHub Personal Access Token |
| `GITHUB_USERNAME` | Required | GitHub account username |
| `STUDENT_SECRET` | Required | Authentication secret for incoming requests |
| `LOG_FILE_PATH` | `logs/app.log` | Location for log file |
| `MAX_CONCURRENT_TASKS` | `2` | Maximum parallel task processing |
| `KEEP_ALIVE_INTERVAL_SECONDS` | `30` | Heartbeat log interval |
| `GITHUB_API_BASE` | `https://api.github.com` | GitHub API endpoint |
| `GITHUB_PAGES_BASE` | Auto-generated | GitHub Pages base URL |

## 🔍 Monitoring & Debugging

### Logging

All operations are logged with timestamps and severity levels:
- **INFO**: Normal operations (task received, files saved, deployments)
- **WARNING**: Recoverable issues (attachment fetch failures, retries)
- **ERROR**: Failed operations
- **EXCEPTION**: Critical failures with stack traces

Logs are written to both console and file, with automatic flushing for real-time monitoring.

### Status Tracking

Use the `/status` endpoint to monitor:
- Last received task details
- Number of active background tasks
- Recent task processing timeline

### Log Retrieval

Fetch recent logs via the `/logs` endpoint:
```bash
curl https://your-service.com/logs?lines=500
```

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Built with [FastAPI](https://fastapi.tiangolo.com/)
- Powered by [Google Gemini AI](https://deepmind.google/technologies/gemini/)
- Deployed via [GitHub Pages](https://pages.github.com/)
- Inspired by the LLM Code Deployment educational framework

## 📞 Support

For issues, questions, or contributions:
- **GitHub Issues**: [Create an issue](https://github.com/dibakash/llm-app-deployer/issues)
- **Repository**: [github.com/dibakash/llm-app-deployer](https://github.com/dibakash/llm-app-deployer)

---

**Built with ❤️ for automated, intelligent web application deployment**
