---
layout: post
title:  "FragmentStateAdapter 에서 itemId 를 다룰 때 주의할 점"
author: uchun
categories: [ ViewPager2, FragmentStateAdapter ]
image: assets/covers/boxes_label.jpg
image_description: "photo by
Raymond Rasmusson on Unsplash"
image_source_url: "https://unsplash.com/photos/7EhAf2dBthg"
featured: true
---

이 글은 `ViewPager2` 에서 `FragmentStateAdapter` 를 쓰면서 경험한 간단한 문제와 원인을 찾아본 경험을 공유합니다. 또한 원인을 찾아보면서 알게 된 내용을 정리하였습니다. (코드 내용을 설명한 글이라 코드를 직접 보는 편을 추천합니다)

이 글은 ViewPager2 1.0.0 기반으로 작성되었습니다.


### 다루지 않을 내용

- `ViewPager2` 는 이미 설명된 글이 많이 있어 여기서는 다루지 않겠습니다.
- `Fragment` 와 `FragmentManager` 에 대해서 다루지 않습니다.
- `ViewPager` (ViewPager1)에 대해서도 다루지 않습니다.


### 이 글에서 다룰 내용

- `ViewPager2`에서 `FragmentStateAdapter`를 사용하면서 겪은 간단한 문제 및 해결 방법에 대해서 다룹니다.
- 문제의 원인을 알아보면서 살펴본 `FragmentStateAdapter` 에서 Fragment 가  추가 및 삭제되는 과정 중 문제와 연관된 부분을 간단히 정리해 보았습니다.


### 어느 날 만난 (간단한) 문제

각 page 별 id 가 필요한 화면이 있었습니다. 그래서 `getItemId()` 를 override 해서 해당 page 의 id 를 제공하였습니다.  
처음에는 별다른 문제가 없었지만, 삭제와 추가 기능을 붙이면서 테스트를 해보니 문제가 발견되었습니다.  

FragmentStateAdapter 에서 Fragment 가 삭제되고 추가될 때 Fragment 의 `onSaveInstanceState()` 가 호출되지 않고 destroyView() 가 되고 Fragment 도 destroy 되었습니다.  

간단한 예제를 통해 원인을 찾던 중 `getItemId()` 를 override 하여 사용하다 보면 이와 같은 문제가 발생하는 것을 확인할 수 있었습니다. 또한 화면 회전 시 (viewpager 가 recreate 될 때) crash 가 발생하였습니다.

해당 원인을 찾기 위해 문서와 코드를 확인하던 중 [https://developer.android.com/training/animation/vp2-migration](https://developer.android.com/training/animation/vp2-migration) 에 아래와 같은 문구가 있습니다.

```
Note: The DiffUtil utility class relies on identifying items by ID. If you are using ViewPager2 to page through a mutable collection, you must also override getItemId() and containsItem().
```

FragmentStateAdapter 의 소스의 `getItemId()` 에는 아래와 같은 주석이 있습니다.

```
/**
 * Default implementation works for collections that don't add, move, remove items.
 * <p>
 * TODO(b/122670460): add lint rule
 * When overriding, also override {@link #containsItem(long)}.
 * <p>
 * If the item is not a part of the collection, return {@link RecyclerView#NO_ID}.
 *
 * @param position Adapter position
 * @return stable item id {@link RecyclerView.Adapter#hasStableIds()}
 */
```

위 내용과 같이 `getItemId()` 를 override 하면 `containsItem()` 도 override 해야 합니다.  
제가 겪은 문제의 원인은 `containsItem()` 을 override 하지 않아서였고, override 하여 알맞게 구현해 주니, 문제없이 `onSaveInstanceState()` 가 호출되어 states를 save & restore 할 수 있었습니다.  
또한 recreate 시 crash 가 발생하던 문제도 사라졌습니다.  

추가로 호기심이 발동하여 이전의 ViewPager 에서 사용되는 `FragmentStatePagerAdapter`를 간략히 살펴보니 `FragmentStateAdapter` 와 유사한 형태 입니다만 RecyclerView의 Adapter와 동작 방식이 달라 이와 같은 문제는 발생하지 않을 것으로 보입니다.


### FragmentStateAdapter 에서 Fragment 는 언제, 어떻게 생성되며 제거되는가?

문제는 해결되었지만 아쉬운 느낌이어서 조금 더 보기로 했습니다.  
FragmentStateAdapter 는 RecyclerView.Adapter 를 상속받습니다. 그래서 이번 문제의 원인이 된 onCreateView(), onBindViewHolder() 과정에서 발생하는 일들만 간단히 살펴보겠습니다.


### onCreateViewHolder

```java
@NonNull
@Override
public final FragmentViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
    return FragmentViewHolder.create(parent);
}
```

`onCreateViewHolder()` 에서는 간단히 `FragmentViewHolder` 만 생성합니다.  
`FragmentViewHolder` 는 width, height 가 match_parent 로 된 parent 의 context 를 이용해 생성된  FrameLayout 으로 된 Container를 생성하고 가지고 있으며, FrameLayout을 return 하는 getContainer() method가 있습니다.

나중에 이 container 에 fragment 의 view가 add 되게 됩니다.


### onBindViewHolder

```java
@Override
public final void onBindViewHolder(final @NonNull FragmentViewHolder holder, int position) {
    final long itemId = holder.getItemId();
    final int viewHolderId = holder.getContainer().getId();
    final Long boundItemId = itemForViewHolder(viewHolderId); // item currently bound to the VH
    if (boundItemId != null && boundItemId != itemId) {
        removeFragment(boundItemId);
        mItemIdToViewHolder.remove(boundItemId);
    }

    mItemIdToViewHolder.put(itemId, viewHolderId); // this might overwrite an existing entry
    ensureFragment(position);

    /** Special case when {@link RecyclerView} decides to keep the {@link container}
     * attached to the window, but not to the view hierarchy (i.e. parent is null) */
    final FrameLayout container = holder.getContainer();
    if (ViewCompat.isAttachedToWindow(container)) {
        if (container.getParent() != null) {
            throw new IllegalStateException("Design assumption violated.");
        }
        container.addOnLayoutChangeListener(new View.OnLayoutChangeListener() {
            @Override
            public void onLayoutChange(View v, int left, int top, int right, int bottom,
                    int oldLeft, int oldTop, int oldRight, int oldBottom) {
                if (container.getParent() != null) {
                    container.removeOnLayoutChangeListener(this);
                    placeFragmentInViewHolder(holder);
                }
            }
        });
    }

    gcFragments();
}
```

`onBindViewHolder()` 에서 보이는 두 가지 id 인 `itemId` 와 `viewHolderId` 가 있습니다.

먼저 `itemId` 는  
- holder.getItemId() 를 통해서 얻어오며
- holder의 itemId 는  RecyclerView.Adapter 의 mHasStableIds 가 true 일 때 bindViewHolder 타이밍에 getItemId() 을 통해서 설정됩니다.
- 그리고 FragmentViewHolder 는 생성자에서 setHasStableIds 를 true 로 세팅하고 있습니다.
- 위의 내용으로 보아 itemId == RecyclerView.Adapter 의 getItemId() 를 말합니다.

`viewHolderId` 는  
- `FragmentViewHolder` 의 container 의 id 이며
- 이는 FragmentViewHolder.create 시에 ViewCompat.generateViewId() 값으로 설정됩니다.

itemForViewHolder() 는 viewHolderId 에 매칭 된 itemId 를 찾아주며, bind 될 때마다 viewHolder 에 할당된 itemId 를 점검하며 정리 및 매칭 키를 갱신합니다. RecyclerView 이어서 필요한 로직이며 본 주제와는 거리가 있어 넘어가도록 하겠습니다. (더 궁금하신 분은 코트를 보시면 금방 이해할 수 있는 간단한 로직입니다.)  
그리고 `ensureFragment()` 를 호출하게 됩니다.

FragmentStateAdapter 에서 `getItemId()` 는 기본적으로 단순 position 을 그대로 return 하고 `containsItem()` 은 0 ≤ itemId < itemsize 만 체크하고 있습니다. 그래서 `getItemId()` 를 override 시 `containsItem()` 을 필히 override 해서 구현해 주어야 합니다.


### ensureFragment

```java
private void ensureFragment(int position) {
    long itemId = getItemId(position);
    if (!mFragments.containsKey(itemId)) {
        // TODO(133419201): check if a Fragment provided here is a new Fragment
        Fragment newFragment = createFragment(position);
        newFragment.setInitialSavedState(mSavedStates.get(itemId));
        mFragments.put(itemId, newFragment);
    }
}
```

ensureFragment() 는 해당 위치의 itemId 를 기반으로 mFragments 에 해당 itemId 에 해당하는 Fragment 가 없으면 createFragment() → setInitialSavedState() → 하여 mFragments 에 put 하게 됩니다.

binding 하기 전 해당 Fragment 가 준비되어있는지 확인해주는 역할을 합니다.


### 다시 onBindViewHolder 로 돌아와서

그리고 viewHolder 의 container 에 addOnLayoutChangeListener() 를 등록해 해당 이벤트 발생 시 한 번만 placeFragmentInViewHolder() 로 이어지게 해 줍니다.

하지만 이 작업은 해당 이벤트 발생 시 이뤄질 것이라 그 아래에 있는 `gcFragments()` 가 먼저 호출되게 될 것입니다.


### gcFragments 에서는

mFragment 로 있는 id 중 `containsItem()` 기준에 부합하지 않으면 removeFragment()를 이용하여 정리합니다. 이름 그대로의 기능을 한다고 볼 수 있습니다.  
즉 여기서 getItemId() 와 containsItem() 가 매칭 되지 않으면 원치 않게 Fragment 가 정리되게 됩니다.


### placeFragmentInViewHolder

위에서 본 ensureFragment() 과정에서 준비된 Fragment 를 mFragments 에서 가져와 getView() 를 통해 view 를 얻은 후 아래의 조건에 맞춰 container(위에서 만든 `FragmentViewHolder.getContainer()`) 에 add 하게 됩니다.

```
/*
possible states:
- fragment: { added, notAdded }
- view: { created, notCreated }
- view: { attached, notAttached }

combinations:
- { f:added, v:created, v:attached } -> check if attached to the right container
- { f:added, v:created, v:notAttached} -> attach view to container
- { f:added, v:notCreated, v:attached } -> impossible
- { f:added, v:notCreated, v:notAttached} -> schedule callback for when created
- { f:notAdded, v:created, v:attached } -> illegal state
- { f:notAdded, v:created, v:notAttached } -> illegal state
- { f:notAdded, v:notCreated, v:attached } -> impossible
- { f:notAdded, v:notCreated, v:notAttached } -> add, create, attach
 */
```

gcFragments() 에서 원치 않게 제거 된 Fragment 를 여기서 쓰려고 하면 문제가 발생하게 됩니다. 그 외 illegal state 한 부분에서도 예외가 발생하게 됩니다.


### removeFragment

```java
private void removeFragment(long itemId) {
    Fragment fragment = mFragments.get(itemId);

    if (fragment == null) {
        return;
    }

    if (fragment.getView() != null) {
        ViewParent viewParent = fragment.getView().getParent();
        if (viewParent != null) {
            ((FrameLayout) viewParent).removeAllViews();
        }
    }

    if (!containsItem(itemId)) {
        mSavedStates.remove(itemId);
    }

    if (!fragment.isAdded()) {
        mFragments.remove(itemId);
        return;
    }

    if (shouldDelayFragmentTransactions()) {
        mHasStaleFragments = true;
        return;
    }

    if (fragment.isAdded() && containsItem(itemId)) {
        mSavedStates.put(itemId, mFragmentManager.saveFragmentInstanceState(fragment));
    }
    mFragmentManager.beginTransaction().remove(fragment).commitNow();
    mFragments.remove(itemId);
}
```

itemId 기반으로 `mFragments` 에 있는 Fragment 를 지우고, 지우면서, FragmentManager 의 saveFragmentInstanceState 를 통해 지우는 Fragment 의 state 를 `mSavedStates` 에 보관하고 Fragment 를 정리하는 작업을 합니다.

내용에서 보시는 것 과 같이. `containsItem()` 를 구현해두지 않으면 `mSavedStates` 에 put 되지 않는 것을 알 수 있습니다.


### 정리하며

앞에서 본 것과 같이 `FragmentStateAdapter` 를 상속받아 Adapter 를 만들 때 `getItemId()` 만 override 하고 `containsItem()` 를 override 하지 않으면

1. Fragment 가 remove 될 때 savedState 관리가 되지 않아 ViewPager 에서 Fragment 가 사라지고 다시 나타나면서 destroy 되고 다시 create 될 때 `onSaveInstanceState()` 가 보장되지 않습니다.
2. `gcFragments()` 가 호출될 때 원치 않게 `Fragment` 가 mFragments 에서 지워지며 이로 인해 ViewPager 가 있는 View가 recreate 될 경우 `placeFragmentInViewHolder()` 에서 **IllegalStateException 가 발생하게 됩니다.**
