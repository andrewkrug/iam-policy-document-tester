# iam-policy-document-tester

**Create short-lived, temporary roles for experimenting with AWS IAM policy documents.**

This is a Python function I wrote that lets you rapidly test and experiment with AWS IAM policy documents.
It's best illustrated with an example:

```python
from iam_tester import (
    create_aws_client_from_credentials,
    temporary_iam_credentials
)


admin_role_arn = "arn:aws:iam::1234567890:role/administrator"

policy_document = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:List*",
            "Resource": "*"
        },
        {
            "Effect": "Deny",
            "Action": "s3:List*",
            "Resource": "arn:aws:s3:::my-sekrit-bucket"
        },
    ],
}

with temporary_iam_credentials(admin_role_arn, policy_document) as credentials:
    s3_client = create_aws_client_from_credentials("s3", credentials=credentials)

    # try to list the contents of an S3 bucket, check it's allowed
    # try to list the contents of sekrit-bucket, check it's not allowed
```

I have an IAM policy document that I'd like to test.
In particular, I want to give somebody permissions to list everything in S3 *except* in a secret bucket.

This example will create a temporary IAM role, attach those permissions, then hand me some credentials to use that role.
I can make the API calls I want to test, check they behave correctly, and then the role and associated policies get cleaned up automatically.
The role only exists for the length of the test.

**Epistemic status:** lightly tested, shared as an interesting experiment rather than something you should rely on in production.



## Interesting ideas: what did I learn?

Here are some of the interesting ideas in the code:

*   **Context managers are great for temporary resources.**
    Context managers are a useful Python feature that let you create a resource, and ensure it gets cleaned up afterwards.
    An example you've probably used is the `open` function for files:

    ```python
    with open('spam.txt', 'r') as f:
        print(f.read())
    ```

    The file will be closed when you're done, even if an exception is thrown inside the `with` block.

    I'm using [contextlib.contextmanager](https://docs.python.org/3/library/contextlib.html#contextlib.contextmanager) to create a couple of context managers for temporary IAM resources.
    It looks something like:

    ```python
    import contextlib

    @contextlib.contextmanager
    def temporary_iam_resource(*args, **kwargs):
        # Code to create resource
        resource = create_iam_resource(*args, **kwargs)
        try:
            yield resource
        finally:
            # Code to clean up resource
            delete_iam_resource(resource)
    ```

    This ensures the IAM roles and policies I create always get cleaned up when I'm done.

*   **ExitStack is a good way to handle nested context managers.**
    This script creates several temporary resources, and the nested context managers start to get unwieldy:

    ```python
    with temporary_iam_role() as role1:
        with temporary_iam_role() as role2:
            with temporary_iam_role_policy(role1, policy_document):
                with temporary_iam_role_policy(role2, another_policy_document):
                    ...
    ```

    I recently read about ExitStack in [a blog post by Nikolaus Rath](https://www.rath.org/on-the-beauty-of-pythons-exitstack.html), which gives a way to nest context managers in a cleaner way:

    ```python
    with contextlib.ExitStack() as es:
        role1 = es.enter_context(temporary_iam_role())
        role2 = es.enter_context(temporary_iam_role())
        es.enter_context(temporary_iam_role_policy(role1, policy_document))
        es.enter_context(temporary_iam_role_policy(role2, another_policy_document))
    ```

    Not only does it reduce the amount of indentation required, it also lines things up vertically so it's easier to see the similarities between different lines.

*   **Changes in IAM take a while to propagate.**
    In particular, there's a delay between creating an IAM role, and that role being available to use.
    There's a hard-coded 15 second delay between creating a role and trying to use it, because that's what it took in my testing.

    This isn't a surprise -- IAM is a global, distributed system, and changes won't propagate instantly -- but it's the first time I've encountered it, because I don't usually create roles and immediately try to use them.

*   **You can use EC2's [DescribeRegions API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeRegions.html) to test IAM credentials.**
    The API call has a `DryRun` flag, which tells you if the request was authorised without actually making it, and I saw several examples that suggested using it.

    I did try it here, but it wasn't a reliable source of *"is this role ready yet?"*
    Sometimes a DescribeRegions call would succeed, then the next call would fail, then the next call would succeed.
    Consistency in distributed systems is hard.



## Motivation: why did I write this?

I work on an [archival storage service](https://stacks.wellcomecollection.org/building-wellcome-collections-new-archival-storage-service-3f68ff21927e), which keeps a copy of every object in two S3 buckets (our "permanent storage").
It's important that objects in these buckets are never inadvertently modified or deleted.

Developers have several IAM roles that we use, which give us different permissions within the account (e.g. *read-only*, *billing*, *developer*, *admin*).
Although the latter two roles can usually do almost anything in an account, we have [a blanket "Deny" rule](https://github.com/wellcomecollection/storage-service/blob/95e56ae99498e7f6f8d4a3cb430ba4c318d6f645/terraform/critical_prod/delete_protection.tf#L51-L76) that prevents us from modifying anything in these permanent storage buckets -- so we can't corrupt the archive by accident.

However, sometimes we do want to delete objects -- for example, objects that were stored in the wrong place.

When we do this, I don't want to remove the blanket "Deny" rule, because that puts the archive at higher risk -- including objects that I don't want to change.
Instead, I wanted to create a fine-grained rule that said *"let us delete these three objects, but nothing else"*.

A "Deny" always beats an "Allow" in an IAM policy document, so I can't modify our developer roles to give us these permissions -- instead, I wrote this function to create a *temporary* role with these permissions, which we could then assume to run the deletion.
There's less risk of accidentally deleting something that we weren't planning to delete, because there shouldn't be an IAM policy that allows its deletion.

It's not until I finished that I realised this could be more general-purpose, and used to experiment with IAM policy documents.



## Usage: how can somebody else use this?

Read the code in [iam_tester.py](iam_tester.py), then the working example in [example.py](example.py).

There are probably some hidden assumptions about how we use IAM roles at Wellcome, but you might get it working.



## License

MIT.