---
layout: post
title:  "Fragment 에서 ViewLifecycleOwner 사용 시 주의점"
author: uchun
categories: [ Fragment, LifecycleOwner, LiveData, ]
image: assets/covers/recycle.jpg
image_description: "photo by Lacey Williams on Unsplash"
image_source_url: "https://unsplash.com/photos/Jwh_k0K_QOM"
featured: true
---

### 시작하기에 앞서

이 글은
[http://pluu.github.io/blog/android/2020/01/25/android-fragment-lifecycle/](http://pluu.github.io/blog/android/2020/01/25/android-fragment-lifecycle/) 에 이어서 작성되었습니다. (이하 `이전 글` 로 부르겠습니다)<br>
위 링크의 글을 안 보신 분이면 먼저 보시는 걸 권장합니다.

`이전 글`은 `androidx.Fragment 1.2.0` (이하 Fragment + 버전) 에 기반하여 이야기되었고.<br>
이 글 작성기준 `Fragment 1.2.1`에서 테스트하였습니다.<br>
여기서 다루는 내용에서 이 패치에 해당하는 특별한 차이는 발견하지 못했습니다<br>
그리고 마지막에는 `Fragment 1.2.2`의 내용도 일부 다룹니다.

### ViewLifecycleOwner

이전 글은 Fragment 의 두 가지 Lifecycle 에 대해서 소개했습니다.
이 두 가지의 Lifecycle 을 이용해서 LiveData 에 Observer 를 추가할 때 어떤 문제가 생길 수 있는지 이야기하였습니다.

그러면 LiveData 에 Observer 를 추가할 때 ViewLifecycleOwner 를 언제나 써도 되는 걸까요?<br>
대부분의 경우에는 문제가 되지 않을 것이라 생각합니다.

하지만 onCreate 에서나 constructor 에서 사용해도 될까요?<br>
제 생각에는 그렇게 사용하지 않을 것이라 보지만, refactoring 하다 보면 실수 할 수 있습니다. 그 경우에는 아래의 코드에서 보시는 것처럼 IllegalStateException 가 발생하게 됩니다.

```java
// androidx.fragment.app.Fragment.java

@MainThread
@NonNull
public LifecycleOwner getViewLifecycleOwner() {
    if (mViewLifecycleOwner == null) {
        throw new IllegalStateException("Can't access the Fragment View's LifecycleOwner when "
                + "getView() is null i.e., before onCreateView() or after onDestroyView()");
    }
    return mViewLifecycleOwner;
}
```

그러면 ViewLifecycleOwner 는 언제 생성 및 할당되며 언제 null 처리될까요?

아래의 코드를 보면 performCreateView 에서 onCreateView 호출 이전에 FragmentViewLifecycleOwner 가 생성됩니다.<br>
그러므로 onCreateView 에서 ViewLifecycleOwner 를 사용해도 문제가 없습니다.

```java
// androidx.fragment.app.Fragment.java

void performCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
        @Nullable Bundle savedInstanceState) {
    mChildFragmentManager.noteStateNotSaved();
    mPerformedCreateView = true;
    mViewLifecycleOwner = new FragmentViewLifecycleOwner();
    mView = onCreateView(inflater, container, savedInstanceState);
    if (mView != null) {
        // Initialize the view lifecycle
        mViewLifecycleOwner.initialize();
        // Then inform any Observers of the new LifecycleOwner
        mViewLifecycleOwnerLiveData.setValue(mViewLifecycleOwner);
    } else {
        if (mViewLifecycleOwner.isInitialized()) {
            throw new IllegalStateException("Called getViewLifecycleOwner() but "
                    + "onCreateView() returned null");
        }
        mViewLifecycleOwner = null;
    }
}
```

실제로 위의 mViewLifecycleOwner 의 선언 부를 쫒아가보면 아래와 같이 주석이 달려있습니다.

```java
// androidx.fragment.app.Fragment.java

// This is initialized in performCreateView and unavailable outside of the
// onCreateView/onDestroyView lifecycle
@Nullable FragmentViewLifecycleOwner mViewLifecycleOwner;
MutableLiveData<LifecycleOwner> mViewLifecycleOwnerLiveData = new MutableLiveData<>();
```

그러면 Nullable 한 ViewLifecycleOwner 는 언제 null 이 되는 걸까요?
아래의 소스 코드와 같이 destroyFragmentView 에서 null 이 됩니다.

```java
// androidx.fragment.app.Fragment.java

private void destroyFragmentView(@NonNull Fragment fragment) {
    fragment.performDestroyView();
    mLifecycleCallbacksDispatcher.dispatchOnFragmentViewDestroyed(fragment, false);
    fragment.mContainer = null;
    fragment.mView = null;
    // Set here to ensure that Observers are called after
    // the Fragment's view is set to null
    fragment.mViewLifecycleOwner = null;
    fragment.mViewLifecycleOwnerLiveData.setValue(null);
    fragment.mInLayout = false;
}
```

하지만 우리에게 바로 와 닿는 타이밍은 아니니 조금 더 살펴보면 fragment 에 performDestroyView 가 호출되고
performDestroyView 에서 onDestroyView 가 호출됩니다.<br>
그러므로 onDestroyView 가 끝난 이후에 mViewLifecycleOwner 가 정리되게 됩니다.

### ViewLifecycleOwnerLiveData

Fragment lifecycle 에 이어 ViewLifecycle 까지 신경 써야 하고 ViewLifecycle 는 사용 시 타이밍에도 신경 써야 하니 번거롭게 느껴질 수 있습니다.<br>
그럴 때 ViewLifecycleOwnerLiveData 를 이용할 수도 있습니다.
앞에서 본 소스코드 처럼 ViewLifecycleOwner 가 바뀔 때 마다 ViewLifecycleOwnerLiveData 에 전달되므로 편리하게 쓸 수 있습니다.

제 경우에는 ViewModel 의 LiveData 들을 onCreateView 나 onViewCreated 에서 Observer 를 추가하고 있는데요.
ViewLifecycleOwnerLiveData 를 사용하여 Fragment view 의 생성과정을 떠나  순수하게 ViewLifecycleOwner 만 바라보고 관련된 작업을 분리해 볼 수 있습니다.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    viewLifecycleOwnerLiveData.observe(this, Observer {  })
}
```

이렇게 써 볼 수 있습니다.

ViewLifecycleOwner 가 바뀔 때마다 ViewLifecycle 관련된 작업을 해줄 수 있습니다.

하지만 observe 하려는 LiveData 가 여러 개라면 Observer 를 추가하는 로직을 따로 분리할지도 모릅니다.

예를 들면 아래의 코드와 같이 말이죠.

```kotlin
private fun onViewLifecycleOwnerChanged(viewLifecycleOwner: LifecycleOwner) {
    viewModel.message.observe(viewLifecycleOwner, Observer {
        messageView.text = it
    })
}
```

이와 같은 코드를 ViewLifecycleOwnerLiveData 에 적용해보면

```kotlin
viewLifecycleOwnerLiveData.observe(this, ::onViewLifecycleOwnerChanged)
viewLifecycleOwnerLiveData.observe(this, Observer { onViewLifecycleOwnerChanged(it) })
```

이렇게 써볼 수 있습니다.

하지만 이렇게 되면 Fragment 의 onCreate 에서

```kotlin
onViewLifecycleOwnerChanged(this) // this == Fragment == LifeCycleOwner
```

이 같은 방법으로 호출해도 문제없이 빌드 및 실행이 되어 코드 유지보수 간에 실수할 여지가 생깁니다.

어떤 방식으로 LiveData 를 observe 해야 할지는 같이 작업하는 구성원들과 합의점을 찾을 필요가 있다고 보입니다.

### Lint / UnsafeFragmentLifecycleObserverDetector

이쯤 되면 Lint에 도움을 받을 수 있지 않을까? 싶습니다.

`이전 글`에서 언급된 UnsafeFragmentLifecycleObserverDetector 에 대해 이야기 보면
[https://android.googlesource.com/platform/frameworks/support/+/androidx-master-dev/fragment/fragment-lint/src/main/java/androidx/fragment/lint/UnsafeFragmentLifecycleObserverDetector.kt](https://android.googlesource.com/platform/frameworks/support/+/androidx-master-dev/fragment/fragment-lint/src/main/java/androidx/fragment/lint/UnsafeFragmentLifecycleObserverDetector.kt)

onCreateView(), onViewCreated(), onActivityCreated() 에서 LiveData 에 Observer 추가 시 this 를 써서 ViewLifecycleOwner 가 아닌 Fragment 의 LifecycleOwner 를 이용하면 경고를 해줍니다.

또한 위 3가지 상황에서 호출된 function 에서도 체크를 해줍니다.
아까 예를 든 onViewLifecycleOwnerChanged 와 유사하게 아래와 같은 코드를 작성하면

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    observeData()
}

private fun observeData() {
    viewModel.message.observe(this, Observer {
        messageView.text = it
    })
}
```

observeData 내부에서 Observer 추가 시 LifecycleOwner 로 this 를 이용해도 잘 체크가 됩니다.

하지만 위에 있는 onViewLifecycleOwnerChanged 처럼 lifecycleOwner 를 전달받으면 체크가 되지 않습니다. 이런 패턴도 체크되기를 기대하며, 추가로 체크되지 않는 경우를 소개할까 합니다.

### LiveData extension function `LiveData<T>.observe`

lifecycle-livedata-core-ktx:2.2.0 에서 추가된<br>
Extension function 인 `LiveData<T>.observe` 입니다.

```kotlin
viewModel.message.observe(viewLifecycleOwner, Observer {
    messageView.text = it
})
```

이렇게 LiveData 에 Observer 를 추가 할 수 있는데

```kotlin
viewModel.message.observe(viewLifecycleOwner) {
    messageView.text = it
}
```

이렇게 람다식 으로 표현가능하게 해주는 간단하지만 편리한 extension 입니다.

하지만 아래와 같은 상황에서 lint 가 체크되지 않습니다.

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    viewModel.message.observe(this) {
        messageView.text = it
    }
}
```

앞에서 얘기한 대로 ViewLifecycleOwner 를 사용해야 하는 상황임에도 Fragment 의 Lifecycle 을 사용해 버렸습니다.

extension function 을 쓰지 않으면 되는 상황이지만 람다식이 주는 편리함을 포기하기에는 아쉬웠습니다.
그래서 위의 UnsafeFragmentLifecycleObserverDetector 를 분석해보고<br>
[https://issuetracker.google.com/issues/148996309](https://issuetracker.google.com/issues/148996309) 와 같이 google 에 issue 리포팅 및 개인적으로 수정해본 방법을 제안해 보았습니다.

다행히 제 제안이 잘 받아들여졌고<br>
[https://android-review.googlesource.com/c/platform/frameworks/support/+/1231871/](https://android-review.googlesource.com/c/platform/frameworks/support/+/1231871/) 와 같이 수정 및 반영되었습니다.<br>
`Fragment 1.2.2` 에 반영될 예정이라 합니다.

이번 글에서는 ViewLifecycle 을 쓰면서 조심해야 하는 부분과 고민되는 부분에 대해서 공유해 보았습니다.

아직은 확실히 어떤 방법이 좋다고 말하기 어렵다고 생각합니다.<br>
계속해서 `Fragment` 의 발전되는 모습을 기대해 봅니다.

--- 2020년 2월 20일 추가 ---<br>
2020년 2월 19일 `Fragment 1.2.2` 버전이 릴리즈 되었습니다.<br>
위에서 이야기 한 Lint 이슈 수정사항이 반영되었습니다.<br>
[https://developer.android.com/jetpack/androidx/releases/fragment](https://developer.android.com/jetpack/androidx/releases/fragment)
