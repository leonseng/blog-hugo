---
title: "ELI5: Kubernetes Custom Resources"
date: 2021-07-18T14:50:02+10:00
draft: false
tags: ['kubernetes', 'eli5']
categories: ['tech']
---

In this article, I will be using the process of building a house as an example to explain how Kubernetes [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) work.

---

Imagine building a custom home, which is a highly detailed and laborious work, that you decided to just hire a home builder. The builder gives you a form to fill out details such as:

- how many rooms do you need and what size should they be?
- do you want a garage that fits 1, 2 or 3 cars?
- would you like a swimming pool?
- do you want the utilities connected when you move in?

You fill in the form, specifying the details of your dream home, and hand it over to the builder. Months go by, and voila, the house is completed, ready for you to move in. The builder even offers services to ensure any repairs are covered throughout the occupancy period, and the pool is cleaned regularly!

3 months down, after settling in with all the furniture, you now want a beautiful backyard to go with it. Not a problem, send an updated copy of the form you submitted earlier to include landscaping, and the builder hires some landscapers to build you your garden.

You like the house so much that, 6 years down the road, you fill in another form to get another house built, this time with more room for your growing family, and the builder's done it again!

---

Let's now translate the example above into Kubernetes terms:

- A house consists of many smaller, more fundamental components (walls, bricks, doors, swimming pool etc) which are analogous to the **standard resources in Kubernetes - pods, services, persistent volumes etc**.

- Building a house is a complex process, and that's when it is handy to get someone to do all the work for you, in this case the builder, which acts as a **Custom Controller**.

- The form provided by the builder is the **Custom Resource Definition (CRD)**, which defines a template or constraints the builder supports for the houses it builds. And submitting the forms back to the builder with your preferences on how the houses should look is like declaring instances of **Custom Resources**.

- The builder/**customer controller** then handles all the nitty gritty details of setting up the houses/**custom resources**, as well as making sure any modifications are correctly applied to them.

Often, applications are complex mix of resources, which makes it difficult for end users to manage them without deep understanding of how the resources work together. To lower the the barrier of entry, software vendors provide [Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/) which comprises of custom controllers and CRDs, allowing users to manage the applications in a declarative manner. The controllers, written by the vendors, then do the hard work of getting the application to a desired state.

Hopefully the example above helps you visualize how custom resources are used. If you are interested to build your own CRDs and custom controllers, the following resources are good starting points:

- [CRD examples](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Sample controller](https://github.com/kubernetes/sample-controller)
