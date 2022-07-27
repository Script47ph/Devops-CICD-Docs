# Devops CI/CD Documentation with Gitlab CI

## Install GitLab Runner
1. Simply download one of the binaries for your system:

    ```bash
    sudo curl -L --output /usr/local/bin/gitlab-runner "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64"
    ```
2. Give it permissions to execute:
   ```bash
   sudo chmod +x /usr/local/bin/gitlab-runner
   ```
3. Create a GitLab CI user
   ```bash
   sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
   ```
4. Install and run as service:
   ```bash
   sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
    ```

## Create a Project on Gitlab (Simple Go Apps)
1. Create a new project on GitLab

    From web browser create a new project on GitLab with specification :
    Name project: Golang Apps
    Visibility Level : Public
2. Clone a project
    ```bash
    git clone http://gitlab-local.id/[USERNAME_GITLAB]/golang-apps.git
    cd apps
    ```
3. Create main.go
    ```go
    package main

    import (
        "fmt"
        "os"
    )

    func main() {
        hello_from := os.Getenv("HELLO_FROM")

        if hello_from == "" {
            hello_from = "our GitLab meetup"
        }

        fmt.Println("Hello from " + hello_from + "!")
        fmt.Println("Today we learn about GitLab Best Practices and CI/CD magic :)")

        fmt.Println("");

        fmt.Print(GetTanuki(true))

        fmt.Println("");
        fmt.Println("Join us at https://about.gitlab.com/community/")
    }

    func CreateTmpFile() {
    f, _ := os.Create("tanuki.tmp")
    defer f.Close()

    os.Chmod("tanuki.tmp", 0777)
    }
    ```
4. Create tanuki.go
    ```go
    package main

    import (
    "fmt"
    )

    func Color(colorString string) func(...interface{}) string {
    sprint := func(args ...interface{}) string {
        return fmt.Sprintf(colorString,
        fmt.Sprint(args...))
    }
    return sprint
    }

    var (
    purple = Color("\033[38;5;129m%s\033[0m");
    orange = Color("\033[38;5;208m%s\033[0m");
    yellow = Color("\033[38;5;220m%s\033[0m");
    red = Color("\033[38;5;196m%s\033[0m");
    green = Color("\033[38;5;76m%s\033[0m");
    blue = Color("\033[38;5;27m%s\033[0m");
    )

    func GetTanuki(mode bool) string {
    var out string;

    //Borrowed from https://gitlab.com/gitlab-org/gitlab-foss/-/raw/4086e41fbbcc9d3fe21adf2fb404d778b699d9d7/app/assets/javascripts/console_swag.js
    //Note: Avoid multi line strings in Golang, escaping backticks is a pain.
    out += purple("           +                        +") + "\n";
    out += purple("          :s:                      :s:") + "\n";
    out += purple("         .oso'                    'oso.") + "\n";
    out += purple("         +sss+                    +sss+") + "\n";
    out += purple("        :sssss-                  -sssss:") + "\n";
    out += purple("       'ossssso.                .ossssso'") + "\n";
    out += purple("       +sssssss+                +sssssss+") + "\n";
    out += purple("      -ooooooooo-++++++++++++++-ooooooooo-") + "\n";
    out += red("     ':/") + orange("+++++++++") + yellow("osssssssssssso") + green("+++++++++") + blue("/:'") + "\n";
    out += red("     -///") + orange("+++++++++") + yellow("cssssssssssss") + green("+++++++++") + blue("///-") + "\n";
    out += red("    .//////") + orange("+++++++") + yellow("cosssssssssso") + green("+++++++") + blue("//////.") + "\n";
    out += red("    :///////") + orange("+++++++") + yellow("osssssssso") + green("+++++++") + blue("///////:") + "\n";
    out += red("     .:///////") + orange("++++++") + yellow("ssssssss") + green("++++++") + blue("///////:.'") + "\n";
    out += red("       '-://///") + orange("+++++") + yellow("osssssso") + green("+++++") + blue("/////:-'") + "\n";
    out += red("          '-:////") + orange("++++") + yellow("osssso") + green("++++") + blue("////:-'") + "\n";
    out += red("             .-:///") + orange("++") + yellow("osssso") + green("++") + blue("///:-.") + "\n";
    out += red("               '.://") + orange("++") + yellow("cosso") + green("++") + blue("//:.'") + "\n";
    out += red("                  '-:/") + orange("+") + yellow("coo") + green("+") + blue("/:-'") + "\n";
    out += red("                     '-") + yellow("++") + blue("-'") + "\n";
    out += "\n\n" + red("Tanuki has been summoned...") + "\n";

    return out;
    }
    ```
5. Create go.mod
    ```go
    module gitlab.com/[USERNAME_GITLAB]/golang-apps

    go 1.14

    require github.com/blang/semver/v4 v4.0.0
    ```
6. Push to gitlab
    ```bash
    git add .
    git commit -m "First Commit"
    git branch -M main
    git push -u origin main
    ```
7. Create Dockerfile on new branch "Ex: Develop"
    ```
    FROM golang:latest
    WORKDIR /APP
    ADD . .
    RUN go mod download
    RUN go build -race -ldflags "-extldflags '-static'" -o ./tanuki
    RUN echo "#!/bin/bash \n ./tanuki \n sleep 24h" > start.sh
    RUN chmod +x ./start.sh
    CMD ["./start.sh"]
    ```
8. Create .gitlab-ci.yml
    ```yml
    stages:
        - format
        - test
        - build
        - testing-apps
        - deploy

    image: golang:latest

    variables:
        REPO_NAME: gitlab.com/[USERNAME_GITLAB]/golang-apps
        BINARY_NAME: tanuki
        CONTAINER_IMAGE: [USERNAME_DOCKERHUB]/tanuki

    before_script:
        - mkdir -p $GOPATH/src/$(dirname $REPO_NAME)
        - ln -svf $CI_PROJECT_DIR $GOPATH/src/$REPO_NAME
        - cd $GOPATH/src/$REPO_NAME
        - go mod download

    format:
        stage: format
        script:
            - go fmt $(go list ./... | grep -v /vendor/)
            - go vet $(go list ./... | grep -v /vendor/)
        
    unit-test:
        stage: test
        coverage: '/\d+.\d+% of statements/'
        script:
            - go test -race $(go list ./... | grep -v /vendor/)
            - go test -cover $(go list ./... | grep -v /vendor/)
        
    compile:
        stage: build
        script:
            - go build -race -ldflags "-extldflags '-static'" -o $CI_PROJECT_DIR/$BINARY_NAME
        artifacts:
            paths:
            - $BINARY_NAME
        
    run:
        stage: testing-apps
        script:
            - echo "testing apps. Please wait.."
            - sleep 5s
            - ./$BINARY_NAME
            - sleep 1m
        
    docker-build:
        image: docker:latest
        stage: deploy
        environment: staging
        services:
            - docker:dind
        before_script:
            - echo $DOCKER_TOKEN | docker login -u $DOCKER_USERNAME --password-stdin
        script:
            - docker build -t $CONTAINER_IMAGE:latest .
            - docker push $CONTAINER_IMAGE:latest
    ```
9. Set Variabel CI/CD

    Go to Settings > CI / CD > Variables > Expand then create 2 variables with specification like below.
    - DOCKER_TOKEN = [Token-Dockerhub] (Login to docker hub and generate the access token at this link https://hub.docker.com/settings/security)
    - DOCKER_USERNAME = [Username-Dockerhub]
10. Register GitLab Runner Executor Docker

        Go to Settings > CI / CD > Runners > Shared Runners. Click Disable shared Runners.
        Go to Setting > CI / CD > Runners > Setup a specific Runner manually. Copy the registration token.

    Change the REGISTRATION_TOKEN with Token Runner.

    ```bash
    sudo gitlab-runner register -n \
    --url http://gitlab-local.id \
    --registration-token [REGISTRATION_TOKEN] \
    --executor docker \
    --description "Docker Runner for Go apps" \
    --docker-image "golang:latest" \
    --docker-volumes /var/run/docker.sock:/var/run/docker.sock
    ```
11. Set Permission and Verify the GitLab Runner.
    ```bash
    sudo usermod -aG docker gitlab-runner
    sudo -u gitlab-runner -H docker info
    sudo gitlab-runner start
    sudo gitlab-runner list
    sudo gitlab-runner status
    sudo gitlab-runner verify
    ```
