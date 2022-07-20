# notary
Enforcing image trust on Docker containers using Notary

# What is Notary?
Notary is an implementation of the TUF specification. It is a tool for publishing and managing trusted collections of content. Publishers can digitally sign collections and consumers can verify the integrity and origin of content. This capability is built on a straightforward key management and signing interface to create signed collections and configure trusted publishers.

With Notary, anyone can provide trust over arbitrary collections of data. Using The Update Framework (TUF) as the underlying security framework, Notary takes care of the operations necessary to create, manage, and distribute the metadata necessary to ensure the integrity and freshness of your content. It performs signing of an image using TUF’s roles and keys [hierarchy](https://docs.docker.com/engine/security/trust/).

# What is The Update Framework (TUF)

The Update Framework (TUF) aims to provide a framework (a set of libraries, file formats, and utilities) that can be used to secure new and existing software update systems. These systems can be package managers that are responsible for all of the software that is installed on a system, updaters that are only responsible for individual installed applications, or software library managers that install software that adds functionality such as plugins or programming language libraries. You can find the [full specifications of TUF here](https://theupdateframework.github.io/specification/latest/).

# How to implement image trust in Docker?

We can reduce the attack surface of malicious containers running in your environment by implementing container image trust. With this, we can be sure that only the images you have signed are allowed to run in your environment, thus improving the supply chain security. Docker uses Notary for signing and verifying container images. Let us look at how to enforce container image trust using Docker.

In this demo we will be running the Notary server and Docker registry locally. We will then enable Docker content trust so that we can only pull images from the local Docker registry which are signed by the Notary server.

# How does Notary work?

Let’s start with the core concept of Notary. Notary uses roles and metadata files to sign the content of trusted collections, which are called Globally Unique Names (GUN).

If you would take a Docker image for example, the GUN would be equal to:

![image](https://user-images.githubusercontent.com/88305831/179921294-1b6e5af1-3b16-4be1-9e46-ef567befab1d.png)

[registry] is the URL to your image source repository and [repository name] the name of your image. The [tag] labelling the image (typically with its version).

Notary performs the signing of an image by using TUFs’ roles and key hierarchy. There are five types of keys to sign corresponding metadata files and store them (as .json) in the Notary database[2]. The image below illustrates the key hierarchy and the typical location where the keys are stored:

![image](https://user-images.githubusercontent.com/88305831/179921455-ab3d8351-dc64-4ed9-b4a5-e4f2c0de751a.png)

1. Root Keys: Each GUN has its own root role and key. Root keys are the root of all trust and used to sign the root metadata file that lists the IDs of the root, targets, snapshot and timestamp public keys. The root key is typically held by a collection (GUN) owner and kept offline (e.g. in a local directory such as ~/.docker/trust/private or a Yubikey).

2. Targets Key: The targets key signs the targets metadata file which lists all filenames in the collection, their sizes and respective hashes. The metadata file is used to verify the integrity of all of the actual contents of the repository. This means also that the targets metadata file contains an entry for each image tag. Targets keys can further be used to delegate trust to other collaborators via delegation roles. The targets key is also held by a collection (GUN) owner/administrator and kept locally (e.g. in a local directory such as ~/.docker/trust/private or a Yubikey).

3. Delegation Keys: As stated above targets keys can optionally delegate trust to other delegation roles. These roles will have their own keys to sign delegation metadata files, which list filenames in the collection their sizes and respective hashes. The delegation metadata files are then used to verify the integrity of some or all of the actual contents of the repository. The keys are held by anyone from the collection owner to the collection collaborators (e.g. in a local directory such as ~/.docker/trust/private or a Yubikey).

4. Snapshot Keys: The snapshot key signs the snapshot metadata file, which numerates the filenames, sizes, and hashes of the root, targets, and delegation metadata files for each collection (GUN). The primary objective of the metadata file is to verify the integrity of the other metadata files. The snapshot key is held by either the collection owner (locally) or the Notary service if you use multiple collaborators via delegation roles.

5. Timestamp Keys: The timestamp key signs the timestamp metadata file, which provides freshness guaranties for the collection. It has the shortest expiry time of any particular piece of metadata and by specifying the filename, size, and hash of the most recent snapshot for the collection. The metadata file is used to verify the integrity of the snapshot file. The timestamp key is held by the Notary service so that it can be automatically re-generated when requested by the server.

The actual Notary service to manage the key architecture consists of:

1. A Notary Server that stores and updates the signed metadata files for your trusted collections (GUNs) in an associated database.

2. A Notary Signer which stores private keys to sign metadata for the Notary Server.

The diagram from the Docker documentation of Notary pretty much summarizes the communication between a client and both the Notary Signer and Server. 

Below we adapted a simplified version of that flow.

![image](https://user-images.githubusercontent.com/88305831/179922277-75e0ef37-878f-4ee9-b232-a41880833674.png)


1. Notary Server optionally supports authentication from clients using JWT tokens. If you don’t have the feature enabled you can simply upload new metadata files. When clients upload new metadata files, the Notary Server checks them against any previous versions for conflicts, and verifies the signatures, checksums, and validity of the uploaded metadata.

2. Once all the uploaded metadata has been validated, Notary Server generates the timestamp (and in case of delegations the snapshot) metadata. It sends this generated metadata to the Notary Signer for signing.

3. Notary Signer retrieves the necessary encrypted private keys from its database if available, decrypts the keys, and uses them to sign the metadata. If successful, it sends the signatures back to Notary Server.

4. Notary Server is the source of truth for the state of all trusted collections (GUNs) storing both client-uploaded and server-generated metadata in the TUF database. The generated timestamp and snapshot metadata certify that the metadata files the client uploaded are the most recent for that trusted collection. Notary Server notifies the client that their upload was successful.

5. The client can now immediately download the latest metadata from the server. Notary Server only needs to obtain the metadata from the database, since none of the metadata has expired.

In the case that the timestamp has expired, Notary Server would go through the entire sequence where it generates a new timestamp, request Notary Signer for a signature, and stores the newly signed timestamp in the database. It then sends this new timestamp, along with the rest of the stored metadata, to the requesting client.

Even though signing with Notary may seem complex, the biggest advantage of using Notary to sign your images is probably the existing integration in the Docker client. You can easily enforce image trust on your local devices by using the flags:

* DOCKER_CONTENT_TRUST=1 #to activate Notary on your client and,
 
* DOCKER_CONTENT_TRUST_SERVER=”<url-to-your-Notary-server>” #to provide your own Notary installation as source of trust.
  
After those flags are set, your Docker Client will verify the signature before every pull and request your signing credentials to sign each build before every push.

Docker HUB even provides its own default DOCKER_CONTENT_TRUST_SERVER=” https://notary.docker.io” which, if content trust is activated, is used to sign your pushed images.
  
If a pulled image is signed, you can simply call the command docker trust inspect <GUN> and check the output such as:

$ "docker trust inspect nginx:latest"
  
![image](https://user-images.githubusercontent.com/88305831/179923518-0c3eaa00-3b06-40e4-bac5-964167dde449.png)

  
Besides using the docker trust command, you can also download the Notary Client to communicate directly with your Notary Server.
  
We can see Notary client installation in later section.
  

---

# Steps to encforce container image trust using Docker:

1. Make sure you have docker and docker-compose installed on your system




