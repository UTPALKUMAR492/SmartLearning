# AWS deployment guide

This project is deployed as two applications:

- **Backend:** AWS Elastic Beanstalk running Node.js and Socket.IO.
- **Frontend:** AWS Amplify hosting the Vite build.
- **Database:** MongoDB Atlas. Do not run MongoDB on the frontend or store production credentials in Git.

This approach keeps the existing `backend/` and `frontend/frontendPage/` structure unchanged.

## Before you start

Install or create accounts for:

- AWS account with billing enabled
- GitHub repository containing this project
- MongoDB Atlas cluster
- Node.js 18 or newer and Git on your computer

Rotate the OpenAI key currently in your local `backend/.env`, then create a new key only in the AWS environment variables. Never commit `.env` files.

## 1. Prepare MongoDB Atlas

1. Create a free Atlas cluster and a database user.
2. In **Network Access**, temporarily add `0.0.0.0/0` while learning. Restrict this to AWS networking later.
3. Copy the Node.js connection string and replace its password.
4. Keep this value ready as `MONGO_URI`.

## 2. Deploy the backend with Elastic Beanstalk

Create an Elastic Beanstalk application and choose:

- Environment tier: **Web server environment**
- Platform: **Node.js**
- Branch: the current supported Node.js version
- Application code: upload a ZIP containing the contents of `backend/`, or connect the GitHub repository and set the source directory to `backend`

The backend already has the required `npm start` script. In the environment configuration, add these variables:

```text
NODE_ENV=production
MONGO_URI=<MongoDB Atlas connection string>
JWT_SECRET=<long random secret>
CLIENT_URL=<Amplify frontend URL>
OPENAI_API_KEY=<rotated OpenAI key, if AI features are used>
ADMIN_EMAIL=<your admin email>
ADMIN_PASSWORD=<strong admin password>
ADMIN_NAME=<admin display name>
```

Do not set `PORT`; Elastic Beanstalk supplies it. After deployment, test:

```text
https://<backend-domain>/health
```

It should return `{"ok":true}`. In Elastic Beanstalk, configure the health check path as `/health` if the console offers that option.

## 3. Deploy the frontend with Amplify

1. Open AWS Amplify and choose **Host web app**.
2. Connect the GitHub repository and select the branch.
3. Set the app root to `frontend/frontendPage`.
4. Use these build settings:

```yaml
version: 1
applications:
  - appRoot: frontend/frontendPage
    frontend:
      phases:
        build:
          commands:
            - npm ci
            - npm run build
      artifacts:
        baseDirectory: dist
        files:
          - '**/*'
      cache:
        paths:
          - node_modules/**/*
```

Add this Amplify environment variable:

```text
VITE_API_URL=https://<backend-domain>/api
VITE_SOCKET_URL=https://<backend-domain>
```

After the first frontend deployment, copy its HTTPS URL into the backend's `CLIENT_URL` variable and redeploy the backend. This is required by CORS.

## 4. Verify the deployment

1. Open the Amplify URL and register or log in.
2. Confirm browser requests use the Elastic Beanstalk URL, not `localhost`.
3. Create a course, upload a small note, and open it as a student.
4. Test chat and quiz submission.
5. Check Elastic Beanstalk logs if an API request fails.

## Important upload limitation

The current application stores videos, notes, thumbnails, and chat files in `backend/uploads/`. Elastic Beanstalk's local disk is not permanent: files can disappear after redeploys or instance replacement. This is acceptable for a first demo, but production use should move uploads to Amazon S3 and store only S3 keys or URLs in MongoDB. This guide intentionally does not change that application behavior.

## Updating the application

Push backend or frontend changes to GitHub. Amplify rebuilds the frontend automatically. Deploy the backend again from Elastic Beanstalk when backend files change, and keep the same environment variables.

## Cost and security checklist

- Stop or terminate AWS environments when not practicing.
- Use HTTPS URLs for both applications.
- Keep `.env` files out of Git.
- Replace the temporary Atlas `0.0.0.0/0` rule when the application is stable.
- Rotate any key that was exposed or pasted into chat, logs, or source control.