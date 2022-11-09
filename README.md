# Project A

This project manages data via DVC and pushes it to a DVC remote using [GitLab's generic packages repository](https://docs.gitlab.com/ee/user/packages/generic_packages/).

## How this project/branch was created

1.  Initialize DVC:

    ```shell-session
    $ dvc init
    Initialized DVC repository.

    You can now commit the changes to git.

    +---------------------------------------------------------------------+
    |                                                                     |
    |        DVC has enabled anonymous aggregate usage analytics.         |
    |     Read the analytics documentation (and how to opt-out) here:     |
    |             <https://dvc.org/doc/user-guide/analytics>              |
    |                                                                     |
    +---------------------------------------------------------------------+

    What's next?
    ------------
    - Check out the documentation: <https://dvc.org/doc>
    - Get help and share ideas: <https://dvc.org/chat>
    - Star us on GitHub: <https://github.com/iterative/dvc>
    ```

1.  Configure the DVC remote:

    ```sh
    dvc remote add gitlab.com:40913719 https://gitlab.com/api/v4/projects/40913719/packages/generic/dvc-data
    dvc remote modify gitlab.com:40913719 method PUT
    dvc remote modify gitlab.com:40913719 auth custom

    # The password is a GitLab deploy token with scope `write_package_registry`.
    dvc remote modify --local gitlab.com:40913719 custom_auth_header 'DEPLOY-TOKEN'
    dvc remote modify --local gitlab.com:40913719 password $GITLAB_DEPLOY_TOKEN
    ```

1.  Create a sample text file:

    ```sh
    echo "hello world" > data.txt
    ```

1.  Track `data.txt` using DVC:

    ```sh
    dvc add data.txt
    ```

1.  Push `data.txt` to the DVC remote:

    ```shell-session
    $ dvc push
    1 file pushed
    ```

1.  Commit the changes and push to the remote Git repository:

    ```sh
    git add .
    git commit
    git push origin project-a
    ```