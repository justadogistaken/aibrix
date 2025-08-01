.. _quickstart:

==========
Quickstart
==========

Installing AIBrix in your Kubernetes Cluster
----------------------------------------------

Install AIBrix Components
~~~~~~~~~~~~~~~~~~~~~~~~~

Get your kubernetes cluster ready, run following commands to install aibrix components in your cluster.

.. note::
    If you just want to install specific components or specific version, please check installation guidance for more installation options.

.. code-block:: bash

    kubectl create -f https://github.com/vllm-project/aibrix/releases/download/v0.3.0/aibrix-dependency-v0.3.0.yaml
    kubectl create -f https://github.com/vllm-project/aibrix/releases/download/v0.3.0/aibrix-core-v0.3.0.yaml

Wait for few minutes and run `kubectl get pods -n aibrix-system` to check pod status util they are ready.

.. code-block:: bash

    NAME                                         READY   STATUS    RESTARTS   AGE
    aibrix-controller-manager-56576666d6-gsl8s   1/1     Running   0          5h24m
    aibrix-gateway-plugins-c6cb7545-r4xwj        1/1     Running   0          5h24m
    aibrix-gpu-optimizer-89b9d9895-t8wnq         1/1     Running   0          5h24m
    aibrix-kuberay-operator-6dcf94b49f-l4522     1/1     Running   0          5h24m
    aibrix-metadata-service-6b4d44d5bd-h5g2r     1/1     Running   0          5h24m
    aibrix-redis-master-84769768cb-fsq45         1/1     Running   0          5h24m


Deploy base model
~~~~~~~~~~~~~~~~~

Save yaml as `model.yaml` and run `kubectl apply -f model.yaml`.

.. literalinclude:: ../../../samples/quickstart/model.yaml
   :language: yaml

Ensure that:

1. The `Service` name matches the `model.aibrix.ai/name` label value in the `Deployment`.
2. The `--served-model-name` argument value in the `Deployment` command is also consistent with the `Service` name and `model.aibrix.ai/name` label.


Invoke the model endpoint using gateway API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Depending on where you deployed the AIBrix, you can use either of the following options to query the gateway.

.. code-block:: bash

    # Option 1: Kubernetes cluster with LoadBalancer support
    LB_IP=$(kubectl get svc/envoy-aibrix-system-aibrix-eg-903790dc -n envoy-gateway-system -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')
    ENDPOINT="${LB_IP}:80"

    # Option 2: Dev environment without LoadBalancer support. Use port forwarding way instead
    kubectl -n envoy-gateway-system port-forward service/envoy-aibrix-system-aibrix-eg-903790dc 8888:80 &
    ENDPOINT="localhost:8888"

.. attention::

    Some cloud provider like AWS EKS expose the endpoint at hostname field, if that case, you should use ``.status.loadBalancer.ingress[0].hostname`` instead.

.. code-block:: bash

    # list models
    curl -v http://${ENDPOINT}/v1/models

    # completion api
    curl -v http://${ENDPOINT}/v1/completions \
        -H "Content-Type: application/json" \
        -d '{
            "model": "deepseek-r1-distill-llama-8b",
            "prompt": "San Francisco is a",
            "max_tokens": 128,
            "temperature": 0
        }'

    # chat completion api
    curl http://${ENDPOINT}/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "deepseek-r1-distill-llama-8b",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "help me write a random generator in python"}
        ]
    }'

.. code-block:: python

    from openai import OpenAI
    
    client = OpenAI(base_url="http://${ENDPOINT}/v1", api_key="OPENAI_API_KEY",
                    default_headers={'routing-strategy': 'least-request'})

    completion = client.chat.completions.create(
        model="deepseek-r1-distill-llama-8b",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "What is the capital of California?"}
        ]
    )
    print(completion.choices[0].message.content)

.. code-block:: go

    # multiturn conversation
    package main

    import (
        "context"

        "github.com/openai/openai-go"
        "github.com/openai/openai-go/option"
    )

    func main() {
        client := openai.NewClient(
            option.WithBaseURL("http://${ENDPOINT}:8888/v1"),
            option.WithAPIKey("OPENAI_API_KEY"),
            option.WithHeader("routing-strategy", "prefix-cache"),
        )
        chatCompletion, _ := client.Chat.Completions.New(context.TODO(), openai.ChatCompletionNewParams{
            Messages: []openai.ChatCompletionMessageParamUnion{
                openai.SystemMessage("You are a helpful assistant."),
                openai.UserMessage("What is the capital of California?"),
            },
            Model: "deepseek-r1-distill-llama-8b",
        })
        println(chatCompletion.Choices[0].Message.Content)

        chatCompletion, _ = client.Chat.Completions.New(context.TODO(), openai.ChatCompletionNewParams{
            Messages: []openai.ChatCompletionMessageParamUnion{
                openai.SystemMessage("You are a helpful assistant."),
                openai.UserMessage("What is the capital of California?"),
                openai.AssistantMessage(chatCompletion.Choices[0].Message.Content),
                openai.UserMessage("What is the largest county of california?"),
            },
            Model: "deepseek-r1-distill-llama-8b",
        })
        println(chatCompletion.Choices[0].Message.Content)
    }


If you meet problems exposing external IPs, feel free to debug with following commands. `101.18.0.4` is the ip of the gateway service.

.. code-block:: bash

    kubectl get svc -n envoy-gateway-system
    NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                   AGE
    envoy-aibrix-system-aibrix-eg-903790dc   LoadBalancer   10.96.239.246   101.18.0.4    80:32079/TCP                              10d
    envoy-gateway                            ClusterIP      10.96.166.226   <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP   10d

Local Development with CPU-only vLLM
------------------------------------

This section explains how to run vLLM in a local Kubernetes cluster using CPU-only environments (e.g., for macOS or Linux dev).

Download model locally
~~~~~~~~~~~~~~~~~~~~~~

Use Hugging Face CLI:

.. code-block:: bash

   huggingface-cli download facebook/opt-125m

Start local cluster with kind
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Edit ``kind-config.yaml`` to mount your model cache, then:

.. code-block:: bash

   kind create cluster --config=./development/vllm/kind-config.yaml

For Dev & Testing Local Setup with Monitoring
---------------------------------------------

.. code-block:: bash

    make dev-install-in-kind
    make dev-port-forward
    make dev-stop-port-forward
    make dev-uninstall-from-kind


Build and load images
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   make docker-build-all
   kind load docker-image aibrix/runtime:nightly

Load CPU environment image
~~~~~~~~~~~~~~~~~~~~~~~~~~

**For macOS:**

.. code-block:: bash

   docker pull aibrix/vllm-cpu-env:macos
   kind load docker-image aibrix/vllm-cpu-env:macos

**For Linux:**

.. code-block:: bash

   docker pull aibrix/vllm-cpu-env:linux-amd64
   kind load docker-image aibrix/vllm-cpu-env:linux-amd64

Deploy vLLM model in kind cluster
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**For macOS:**

.. code-block:: bash

   kubectl create -k development/vllm/macos

**For Linux:**

.. code-block:: bash

   kubectl create -k development/vllm/linux

Access model endpoint
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   kubectl port-forward svc/facebook-opt-125m 8000:8000 &

Query locally:

.. code-block:: bash

   curl -v http://localhost:8000/v1/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer test-key-1234567890" \
     -d '{
        "model": "facebook-opt-125m",
        "prompt": "Say this is a test",
        "temperature": 0.5,
        "max_tokens": 512
      }'

Practical Notes
~~~~~~~~~~~~~~~

- ``vllm-cpu-env`` is ideal for development and debugging. Inference latency will be high due to CPU-only backend.
- Be sure to mount your Hugging Face model cache directory, or the container will re-download it online.
- Confirm both ``runtime`` and ``env`` images are loaded into kind.
- Use ``kubectl logs`` or ``kubectl exec`` to debug model pod issues.

Debugging Gateway IPs
~~~~~~~~~~~~~~~~~~~~~

.. code-block:: bash

   kubectl get svc -n envoy-gateway-system

.. code-block::

   NAME                                     TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                   AGE
   envoy-aibrix-system-aibrix-eg-903790dc   LoadBalancer   10.96.239.246   101.18.0.4    80:32079/TCP                              10d

Please also follow `debugging guidelines <https://aibrix.readthedocs.io/latest/features/gateway-plugins.html#debugging-guidelines>`_.