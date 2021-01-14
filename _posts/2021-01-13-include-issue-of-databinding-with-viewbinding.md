---
layout: post
title:  "FragmentStateAdapter 에서 itemId 를 다룰 때 주의할 점"
author: uchun
categories: [ DataBinding, ViewBinding ]
image: assets/covers/binding.jpg
image_description: "photo by
henry perks on Unsplash"
image_source_url: "https://unsplash.com/photos/9l53r3K7PKU"
featured: fales
---
`DataBinding` 만 쓰다가 `ViewBinding` 을 쓰기 위해서 해당 기능을 활성화하면 include tag 에 관련된 에러를 볼 수 있습니다. 쉽게 수정 가능한 이슈였고 약간의 선택지가 있어 기록을 남겨둡니다.

*본문의 예제는 편의상 필요한 부분만 간략히 표현하였습니다.*

---
### 먼저 가상의 상황을 만들어 보겠습니다.

`DataBinding`만 이용하고 있으며 layout 내부에는 include tag 를 사용했습니다.
```xml
<!-- fragment_main.xml -->
<layout>
	<data>
		<variable name="viewModel" ... />
	</data>
	<XXXLayout>
		<include android:id="@+id/message_view" layout="@layout/message_view" />
		...
	</XXXLayout>
</layout>

<!-- message_view.xml -->
<XXXLayout>
	...
</XXXLayout>
```

include 한 message_view 의 visibility 를 바꾸기 위해 아래와 같이 접근 할 수 있습니다.
```kotlin
val binding: FragmentMainBinding
binding.messageView.visibiliity = View.VISIBLE
```

### 이와 같은 상황에서 ViewBinding 을 활성화하면 몇 가지 추가 작업이 필요합니다.

1. 기존의 DataBinding 으로 사용하고 방식에서 에서 include 에 id 를 부여한 경우 view 로 접근 가능했으나 ViewBinding 을 활성화하면 ViewBinding 으로 취급되어 binding.layoutId 가 아닌 binding.layoutId.root 로 접근해야 합니다.
```kotlin
// before
binding.messageView.visibiliity = View.VISIBLE
// after
binding.messageView.root.visibiliity = View.VISIBLE
```

2. 만약 프로젝트의 Android Gradle Plugin(이하 AGP)의 버전이 4.1 미만이라면
```
error: incompatible types: MessageViewBinding cannot be converted to ViewDataBinding
setContainedBinding(this.messageView);
```
와 같은 에러를 볼 수 있습니다.


### 위의 2번의 에러를 해결하기 위해서는 2가지 방법이 있습니다.

1. DataBinding 을 위해 layout tag 를 사용하는 레이아웃 안에서 include 가 사용된 경우 include 할 layout 을 layout tag 로 감싸줘서 해결할 수 있습니다.
```xml
<!-- before / message_view.xml -->
<XXXLayout>
  ...
</XXXLayout>

<!-- after / message_view.xml -->
<layout>
  <XXXLayout>
    ...
  </XXXLayout>
</layout>
```

2. 다른 방법은 AGP 를 4.1 이상으로 올리면 레이아웃 수정 없이 문제가 해결됩니다. AGP 변경 사항에 따라 추가로 수정이 필요할 수 있으며 자세한 내용은 공식 사이트에서 확인 가능합니다.
  - 영문 : [https://developer.android.com/studio/releases/gradle-plugin](https://developer.android.com/studio/releases/gradle-plugin)
  - 한글 : [https://developer.android.com/studio/releases/gradle-plugin?hl=ko](https://developer.android.com/studio/releases/gradle-plugin?hl=ko)
