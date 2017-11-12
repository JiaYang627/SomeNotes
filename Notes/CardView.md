# CardView

> Android5.0中新增了CardView，一个继承自FrameLayout类、可以设置圆角、阴影的控件，同样也可以包含其他布局容器和控件。

* 配置
如果SDK低于5.0，我们仍要引入 v7 包。在build.gradle中加入如下代码以自动导入support-v7包。

```
dependencies {
    ...
    compile 'com.android.support:appcompat-v7:25.3.1'
    compile 'com.android.support:cardview-v7:25.3.1'
}
```

* 使用
布局如下：

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout

    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">


    <android.support.v7.widget.CardView
        android:id="@+id/cardView"
        android:layout_width="match_parent"
        android:layout_height="250dp"
        app:cardCornerRadius="20dp"
        app:cardElevation="20dp">

        <ImageView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="@drawable/cardview"
            android:scaleType="centerInside"/>

    </android.support.v7.widget.CardView>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="30dp">

        <SeekBar
            android:id="@+id/sb_1"
            android:layout_width="200dp"
            android:layout_height="wrap_content"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="控制圆角大小"/>

    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="30dp">

        <SeekBar
            android:id="@+id/sb_2"
            android:layout_width="200dp"
            android:layout_height="wrap_content"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="控制阴影大小"/>

    </LinearLayout>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="30dp">

        <SeekBar
            android:id="@+id/sb_3"
            android:layout_width="200dp"
            android:layout_height="wrap_content"/>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="控制图片间距"/>

    </LinearLayout>

</LinearLayout>
```
此处有两个CardView的重要属性 ：cardCornerRadius 设置圆角半径，cardElevation 设置阴影半径。

除此以外，CardView还有其他属性：

```
        CardView_cardBackgroundColor:       设置背景色。
        CardView_cardMaxElevation:          设置Z轴最大高度值。
        CardView_cardUseCompatPadding:      是否使用CompatPadding。
        CardView_cardPreventCornerOverlap:  是否使用PreventCornerOverlap。
        CardView_contentPadding:            内容的Padding。
        CardView_contentPaddingLeft:        内容的左Padding。
        CardView_contentPaddingTop:         内容的上Padding。
        CardView_contentPaddingRight:       内容的右Padding。
        CardView_contentPaddingBottom:      内容的底Padding。
```

Java代码中：

```
   @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_cardview);

        mCardView = (CardView) findViewById(R.id.cardView);
        mSeekBar1 = (SeekBar) findViewById(R.id.sb_1);
        mSeekBar2 = (SeekBar) findViewById(R.id.sb_2);
        mSeekBar3 = (SeekBar) findViewById(R.id.sb_3);

        initAssignViews();

    }

    private void initAssignViews() {
        mSeekBar1.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                mCardView.setRadius(progress);
            }

            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {

            }

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {

            }
        });

        mSeekBar2.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                mCardView.setCardElevation(progress);
            }

            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {

            }

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {

            }
        });

        mSeekBar3.setOnSeekBarChangeListener(new SeekBar.OnSeekBarChangeListener() {
            @Override
            public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
                mCardView.setContentPadding(progress, progress, progress, progress);
            }

            @Override
            public void onStartTrackingTouch(SeekBar seekBar) {

            }

            @Override
            public void onStopTrackingTouch(SeekBar seekBar) {

            }
        });


    }
}
```

此处设置了3个SeekBar分别设置CardView的 圆角半径：mCardView.setRadius，阴影半径：mCardView.setCardElevation，内部子父控件的距离：mCardView.setContentPadding。


具体Demo:[](https://github.com/JiaYang627/GitHubNotesDemo)
