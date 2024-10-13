---
layout: post
title:  "Strapi: Register User"
date:   2024-10-13 16:09:27 +0200
categories: strapi 
---
## What we cover

Today, we implement a User Registration in the Headless CMS Strapi. Strapi has a plugin integrated called *user-permissions*, which can handle the access to the Strapi Data and includes an Authentication Logic with Password forgotten/Reset Password and a Role Management.

We will use:

- Strapi V4
- VS Code Plugin: Thunder Client
- next JS 14 with Typescript
- Apollo GraphQL

We will not cover how GraphQL and Typescript are used.

As the user in the plugin users-permissions is restricted to the fields username, email, password, we use a new collection (e.g. customer), which holds the rest of the data and has a relation to the user.

## Creating the Apollo Client

After installing the basics (i.e. strapi and next), we need to set up the apollo client: As we will create a user in strapi, we will use a privileged user with access rights to create a user in the database. Create one in the strapi backend and assign him a new role. This role should now be allowed to create users with the users-permissions plugin.

We create an async function, which will return a json web token (JWT). This token will queried and needed with every future mutation of strapi.

```tsx
export async function authenticateUser(email: string, password: string) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_STRAPI_URL}/api/auth/local`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        identifier: email, // E-Mail or Username
        password: password,
      }),
    },
  );

  if (!response.ok) {
    const errorData = await response.json();
    throw new Error(
      `Authentification failed: ${errorData.error.message}`,
    );
  }

  const data = await response.json();
  return data.jwt;
}

```

No we can add this JWT in the authorization header of the apollo client:

```tsx
import { ApolloClient, InMemoryCache, HttpLink } from "@apollo/client";
import { authenticateUser } from "./authenticateUser";
import fetch from "cross-fetch";

/**
 * Creates an Apollo Client with the given email and password.
 * If no email and password are provided, the client will be created without authentication.
 * @param {string | undefined} email - The email of the user.
 * @param {string | undefined} password - The password of the user.
 * @returns {ApolloClient<InMemoryCache>} The Apollo Client.
 */
export async function createApolloClient(email = "", password = "") {
  let jwtToken;
  let headers = {};
  if (email && password) {
    jwtToken = await authenticateUser(email, password);
    headers = {
      Authorization: `Bearer ${jwtToken}`,
    };
  }

  return new ApolloClient({
    ssrMode: true,
    link: new HttpLink({
      uri: `${process.env.NEXT_PUBLIC_STRAPI_URL}/graphql`,
      headers: headers,
      fetch,
    }),
    cache: new InMemoryCache(),
  });
}
```

## Register User

We can now write a mutation to register a user. We further add the additional collection to strapi (it’s called user_prospect here). In the following code, we firstly register the user, get the user id and add this id with the other prospect’s data tu user_prospect.

### Code Explanation:

1. A username is required, so i set the email to the username here.
2. The mutation RegisterUser calls the function register with the parameter input and returns the user with id, email and password.
3. The mutation CreateUserProspect calls the function createUserProspect (although the collection is written with underscore *user_prospect*).
4. publishedAt is needed, otherwise the customer/user_prospect collection is set to draft and is not published, yet. By setting publishedAt to null, it will stay draft.

```tsx
export async function registerInStrapi(formData: Strapi_User_Registration) {
  const client = await createApolloClient(
    process.env.STRAPI_EMAIL,
    process.env.STRAPI_PASSWORD,
  );

  const {
    email,
    email: username,
    password,
    firstname,
    lastname,
    institution,
    position,
    city,
    postal_code,
    has_newsletter,
  } = formData;

  try {
    const { data: registerData } = await client.mutate({
      mutation: gql`
        mutation RegisterUser(
          $username: String!
          $email: String!
          $password: String!
        ) {
          register(
            input: { username: $username, email: $email, password: $password }
          ) {
            user {
              id
              username
              email
            }
          }
        }
      `,
      variables: {
        username,
        email,
        password,
        firstname,
        lastname,
        institution,
        position,
        city,
        postal_code,
        has_newsletter,
      },
    });

    const id = registerData.register.user.id;

    const { data: updateUserProspectData } = await client.mutate({
      mutation: gql`
        mutation CreateUserProspect(
          $firstname: String!
          $lastname: String!
          $institution: String!
          $position: String!
          $city: String!
          $postal_code: String!
          $has_newsletter: Boolean!
          $id: ID!
          $publishedAt: DateTime!
        ) {
          createUserProspect(
            data: {
              firstname: $firstname
              lastname: $lastname
              institution: $institution
              position: $position
              city: $city
              postal_code: $postal_code
              has_newsletter: $has_newsletter
              users_permissions_user: $id
              publishedAt: $publishedAt
            }
          ) {
            data {
              id
            }
          }
        }
      `,
      variables: {
        id,
        firstname,
        lastname,
        institution,
        position,
        city,
        postal_code,
        has_newsletter,
        publishedAt: new Date().toISOString(),
      },
    });

    return updateUserProspectData;
  } catch (error) {
    console.error("Error registering user:", error);
    throw new Error("Registration failed");
  }
}
```

## Final Words (about Thunder Client)

The API debugging is not quite easy when running the functions, so I decided to use Thunder Client while Coding to request GraphQL queries and get a nice error output.