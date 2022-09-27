---
layout: post
title:  "Android onSaveInstanceState() Bundle -- Is It Secret? Is It Safe?"
date:   2016-05-16 -0700
categories:
---

> Is it secret? Is it safe?

&nbsp;- Gandalf

I recall a lesson from a second-year computer science course.


A penetration team was testing a password authentication API. The particular OS hosting the API allowed user-mode code to register a callback to fire whenever a page-fault occurred. The penetration team made the assumption that internally the OS was doing a character-by-character comparison between the user-supplied password and the correct password, and that the comparison loop terminated upon the first unmatched character.


So they aligned the password buffer such that the first character of the password occupied the last character of a page of memory, and proceeded to iterate through the alphabet of *m* possible characters. If the password API failed, then that character was not the first character of the password. If the page-fault callback fired, then that character was the first character in the password. This procedure could be repeated for each character position in the password provided that the previous character positions matched the password. Consequently, what should have required an intractable O(*m^n*)-time attack was reduced to an inexpensive O(*m·n*)-time attack. 

There were a number of lessons to be taken from this example, such as *complete the comparison even if a character fails or even better use a one-way hash instead of character-by-character comparison*.

But what I took from this lesson was a bit more abstract: *it's extremely difficult (if not impossible) to predict the vector of an attack*.

I have since graduated and now find myself writing mobile software. Presently this is iOS and Android, and in the past it was Windows. The code I write is largely focused on enterprise and health care.

What enterprise and healthcare have in common is the need for data security. This breaks down into two broad areas of concern for me: data communication and data persistence. The data communication aspect largely deals with SSL and user authentication – this is all pretty standard.

The data persistence, however, is somewhat more interesting, largely because unlike data communication we really don't have established standards for dealing with it. For example, while SSL is almost universally accepted, iOS sandbox-level encryption and Android file-system encryption are not.

A stolen Android device may be jail-broken and the files stolen, so some clients may say that Android sandboxing and file-system encryption are not adequate for their needs.

Or a user may not know to enable iOS sandbox-level encryption. The first time I presented iOS sandbox-level encryption as a solution to a client the question I was asked was “can we force the customers to use it?” My answer was “no,” to which the client responded “well then we can't use it”.

Consequently when it comes to persistence, I need to look and feel around and anticipate the holes where secure data may be leaked. This typically involves examining the app and it's sandbox directory. What files does it create? Where does it's cache go? Where are it's settings saved? What are all these files I'm seeing in the sandbox?

## And where does this outState bundle go???

`protected void onSaveInstanceState(Bundle outState)`

This is a method on `android.app.Activity` (`android.app.Fragment` has something similar). It gets called by Android when the Activity needs to be destroyed but also needs to be re-instantiated at some point in the near future – let's call this “temporary destruction”. This typically happens when the device is rotated and Android destroys the current Activity and then re-creates it in the new orientation.

The Activity may also be destroyed (temporarily) in order to free up memory. This could happen if the Activity is not visible and a low memory condition occurs.

So from the user's point of view, “temporary destruction” isn't destruction at all. The client code overrides `onSaveInstanceState(Bundle outState)` and saves any UI state that needs to be restored to the new Activity instance to the `outState` bundle. When the Activity is re-instantiated, a copy of the `outState` bundle is passed back in through the `onCreate()` method:

`protected void onCreate(Bundle savedInstanceState)`

So where is this bundle being saved to? Bundles are an [IPC](https://en.wikipedia.org/wiki/Inter-process_communication) mechanism, so it's not going to the filesystem. But now there's a *P* involved – which process is it? And what is that process doing with this data? And do I need to be worried about it?

It turns out that these instance state bundles are stored in the *Activity Manager* service. This service is implemented under the package `com.android.server.am` in the Android source code. Recall that Activities are stacked one on top of another and that Android calls these stacks “Tasks” (don't confuse “Task” with “Process” – in computer science they may mean the same thing but in Android they are very different concepts). Each of these tasks is represented internally with an object of class `TaskRecord`. This class contains an array of `ActivityRecord` objects, each of which manages the state of an Activity. `ActivityRecord` contains a member of type Bundle named *icicle*. This *icicle*-bundle is the saved instance state and it is actually stored in the memory space of the *Activity Manager* service.

## Do I need to be worried?
Recently there was an article on [Ars Technica](https://arstechnica.com/information-technology/2014/03/malicious-apps-can-brick-android-phones-erase-data-researchers-warn/) that described an attack on Android devices in which an app-name of over 387,000 characters could “send vulnerable devices into a spiral of endlessly looping crashes and possibly delete all data stored on them.” This problem was simultaneously predictable and unpredictable. It was predictable because this type of buffer overflow attack is classic and fairly well known (see [Morris Worm](https://en.wikipedia.org/wiki/Morris_worm)). It was (effectively) unpredictable because of the intractability of predicting how a computer program will behave against all inputs (see Alan Turing's [Halting Problem](https://en.wikipedia.org/wiki/Halting_problem)).

We see examples of this almost weekly. ATM and credit card readers are hacked. Laptops with sensitive data stored on unencrypted hard-drives are stolen. Universities, businesses, and government agencies are routinely hacked into.

*Our data has become valuable enough to steal, and sophisticated thieves are thinking of creative new ways to steal it.*

Device theft has become a rampant problem but those devices are only being sold for hard cash. Yet I see the day coming when devices are being stolen and jail-broken just to get at the data they contain. The thieves don't even need the data for themselves – they might just upload it to an underground cloud and sell it to a black market for personal data. It would be the digital equivalent of the dumpster-diving that goes on to facilitate identity theft.

## Am I Worrying too Much?
In Microsoft’s .NET runtime there is a class named `System.Security.SecureString`. This class exists to represent strings in a secure format. The documentation describes it as follows:

> Represents text that should be kept confidential. The text is encrypted for privacy when being used, and deleted from computer memory when no longer needed. This class cannot be inherited.

In the Java security APIs you never see passwords stored in a `String`. Instead they are stored in `char[]`. Why is this? It's because a `String` is immutable, but a `char[]` can be overwritten. This gives the client code the ability to clear out the `char[]` when it is done using the password. This clearing reduces the chance for an attacker to read the password from memory. There is an indeterminate amount of time before the garbage collector runs and the password string is reclaimed. Even if the memory is reclaimed, it may not be cleared until it is re-allocated. As an example, the `javax.crypto.spec.PBEKeySpec` class is used to store a password. It does two things: (1) store the password in a `char[]` and (2) provide a `clearPassword()` method that overwrites the `char[]`.

The .NET and Java examples make a convincing argument: *storing sensitive data in memory is a potential security hole*.

Consequently I feel that saving sensitive data into `onSaveInstanceState()` is a potential security hole. I can't think of any scenarios that would allow a third party access to what my Activity stores using `onSaveInstanceState()` but that doesn't mean it won't ever be a problem.

<!-- April 24, 2014 EDIT - I wrote this blog just days before the Heartbleed bug was publicly disclosed! So no I don't think that I can ever worry too much.  -->

## So what to do?
The simplest solution is don't save any sensitive information at all. If the user is entering a password or credit-card number then don't stack a new Activity on top. Rotations can also be disabled for the Activity.

However, if we do wish to save sensitive information, we have options.

First let's differentiate between passwords and non-password sensitive information (e.g. credit-card numbers, social security numbers).

If we wish to store a password we can store it outside of the Activity but still inside the memory space of our app (likely in a `char[]` member of our data model or our Application-derived class). This solution will work for rotations and will likely work if our Activity is destroyed due to low memory. The value will be lost however if the entire application process is terminated.

If we wish to store non-password sensitive information, then we can encrypt that information before passing it into the `outState` bundle. If our app deals with sensitive information, it means that we have some kind of authentication procedure, likely involving a password. This password can be used to derive a symmetric encryption key using PBKDF2. This key can be used to create a `javax.crypto.Cipher` object (likely using AES/CBC/PKCS5Padding encryption), which in turn can be used to encrypt any sensitive information managed by the Activity. The encrypted data can be stored safely in the `outState` bundle.

## Is Activity.onSaveInstanceState() safe?
I'm not insinuating that it isn't. It's probably perfectly safe. I can't think of any ways of attacking it.

But given the unpredictability and creativity and motivation behind information attacks, it really doesn't hurt to think about what we are storing there and how we are securing it.