SkyPilot
========

What is SkyPilot
----------------

SkyPilot is a framework for running LLMs, AI, and batch jobs on any
cloud, offering maximum cost savings, highest GPU availability, and
managed execution. Its features include:

-  Get the best GPU availability by utilizing multiple resources pools
   across multiple regions and clouds.
-  Pay absolute minimum — SkyPilot picks the cheapest resources across
   regions and clouds. No managed solution markups.
-  Scale up to multiple replicas across different locations and
   accelerators, all served with a single endpoint
-  Everything stays in your cloud account (your VMs & buckets)
-  Completely private - no one else sees your chat history

Install SkyPilot
----------------

We advise you to follow the
`instruction <https://skypilot.readthedocs.io/en/latest/getting-started/installation.html>`__
to install Skypilot. Here we provide a simple example of using ``pip``
for the installation as shown below.

.. code:: bash

   git clone https://github.com/skypilot-org/skypilot.git
   cd skypilot

   # Choose your cloud:

   pip install -e ".[aws]"
   pip install -e ".[gcp]"
   pip install -e ".[azure]"
   pip install -e ".[oci]"
   pip install -e ".[lambda]"
   pip install -e ".[runpod]"
   pip install -e ".[fluidstack]"
   pip install -e ".[cudo]"
   pip install -e ".[ibm]"
   pip install -e ".[scp]"
   pip install -e ".[vsphere]"
   pip install -e ".[kubernetes]"
   pip install -e ".[all]"

or you can install it in this way if you use more than one cloud:

.. code:: bash

   pip install -e ".[aws,gcp]"

After that, you need to verify cloud access with a command like:

.. code:: bash

   sky check

For more information, check the official document and see if you have
setup your cloud accounts corerctly.

Alternatively, you can also use the official docker image with SkyPilot
main branch automatically cloned by running:

.. code:: bash

   # NOTE: '--platform linux/amd64' is needed for Apple silicon Macs
   docker run --platform linux/amd64 \
     -td --rm --name sky \
     -v "$HOME/.sky:/root/.sky:rw" \
     -v "$HOME/.aws:/root/.aws:rw" \
     -v "$HOME/.config/gcloud:/root/.config/gcloud:rw" \
     berkeleyskypilot/skypilot-nightly

   docker exec -it sky /bin/bash

Running Qwen1.5-72B-Chat with SkyPilot
--------------------------------------

1. Start serving Qwen1.5-72B-Chat on a single instance with any
   available GPU in the list specified in
   `serve-72b.yaml <https://github.com/skypilot-org/skypilot/blob/master/llm/qwen/serve-72b.yaml>`__
   with a vLLM powered OpenAI-compatible endpoint:

.. code:: bash

   sky launch -c qwen serve-72b.yaml

2. Send a request to the endpoint for completion:

.. code:: bash

   IP=$(sky status --ip qwen)

   curl -L http://$IP:8000/v1/completions \
       -H "Content-Type: application/json" \
       -d '{
         "model": "Qwen/Qwen1.5-72B-Chat",
         "prompt": "My favorite food is",
         "max_tokens": 512
     }' | jq -r '.choices[0].text'

3. Send a request for chat completion:

.. code:: bash

   curl -L http://$IP:8000/v1/chat/completions \
       -H "Content-Type: application/json" \
       -d '{
         "model": "Qwen/Qwen1.5-72B-Chat",
         "messages": [
           {
             "role": "system",
             "content": "You are a helpful and honest chat expert."
           },
           {
             "role": "user",
             "content": "What is the best food?"
           }
         ],
         "max_tokens": 512
     }' | jq -r '.choices[0].message.content'

Scale up the service with SkyServe
----------------------------------

1. With `SkyPilot
   Serving <https://skypilot.readthedocs.io/en/latest/serving/sky-serve.html>`__,
   a serving library built on top of SkyPilot, scaling up the Qwen
   service is as simple as running:

.. code:: bash

   sky serve up -n qwen ./serve-72b.yaml

This will start the service with multiple replicas on the cheapest
available locations and accelerators. SkyServe will automatically manage
the replicas, monitor their health, autoscale based on load, and restart
them when needed.

A single endpoint will be returned and any request sent to the endpoint
will be routed to the ready replicas.

2. To check the status of the service, run:

.. code:: bash

   sky serve status qwen

After a while, you will see the following output:

::

   Services
   NAME        VERSION  UPTIME  STATUS        REPLICAS  ENDPOINT            
   Qwen  1        -       READY         2/2       3.85.107.228:30002  

   Service Replicas
   SERVICE_NAME  ID  VERSION  IP  LAUNCHED    RESOURCES                   STATUS REGION  
   Qwen          1   1        -   2 mins ago  1x Azure({'A100-80GB': 8}) READY  eastus  
   Qwen          2   1        -   2 mins ago  1x GCP({'L4': 8})          READY  us-east4-a 

As shown, the service is now backed by 2 replicas, one on Azure and one
on GCP, and the accelerator type is chosen to be **the cheapest
available one** on the clouds. That said, it maximizes the availability
of the service while minimizing the cost.

3. To access the model, we use a ``curl -L`` command (``-L`` to follow
   redirect) to send the request to the endpoint:

.. code:: bash

   ENDPOINT=$(sky serve status --endpoint qwen)

   curl -L http://$ENDPOINT/v1/chat/completions \
       -H "Content-Type: application/json" \
       -d '{
         "model": "Qwen/Qwen1.5-72B-Chat",
         "messages": [
           {
             "role": "system",
             "content": "You are a helpful and honest code assistant expert in Python."
           },
           {
             "role": "user",
             "content": "Show me the python code for quick sorting a list of integers."
           }
         ],
         "max_tokens": 512
     }' | jq -r '.choices[0].message.content'

Accessing Qwen1.5 with Chat GUI
---------------------------------------------

It is also possible to access the Qwen1.5 service with a GUI using
`FastChat <https://github.com/lm-sys/FastChat>`__.

1. Start the Chat Web UI:

.. code:: bash

   sky launch -c qwen-gui ./gui.yaml --env ENDPOINT=$(sky serve status --endpoint qwen)

2. Then, we can access the GUI at the returned gradio link:

::

   | INFO | stdout | Running on public URL: https://6141e84201ce0bb4ed.gradio.live

Note that you may get better results to use a different temperature and
top_p value.

Summary
-------

With SkyPilot, it is easy for you to deploy Qwen1.5 on any cloud. We
advise you to read the official doc for more usages and more updates.
Check `this <https://skypilot.readthedocs.io/>`__ out!
