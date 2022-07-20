# notary
Enforcing image trust on Docker containers using Notary

# What is Notary?
Notary is an implementation of the TUF specification. It is a tool for publishing and managing trusted collections of content. Publishers can digitally sign collections and consumers can verify the integrity and origin of content. This capability is built on a straightforward key management and signing interface to create signed collections and configure trusted publishers.

With Notary, anyone can provide trust over arbitrary collections of data. Using The Update Framework (TUF) as the underlying security framework, Notary takes care of the operations necessary to create, manage, and distribute the metadata necessary to ensure the integrity and freshness of your content. It performs signing of an image using TUF’s roles and keys hierarchy.
