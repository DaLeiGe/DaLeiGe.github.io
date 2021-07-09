---
layout: post
title: "Android自定义ViewGroup实现九宫格控件"
date: 2021-07-01 12:00:01.000000000 +09:00
categories: [Android, ViewGroup]
tags: [Android]
typora-root-url: ..
---
<meta name="referrer" content="no-referrer"/>


## 一、简介
最近项目里有个类似微信朋友圈的九图控件的需求，Github找了一下，发现都不太满足需求，我需要单张图片的时候可以按照图片宽高比列在一定范围内自适应，而大多开源项目单张图片也是一个小正方形，所以，干脆自己动手写一个

### 1.1、效果图如下
![ezgif.com-crop.gif](https://upload-images.jianshu.io/upload_images/7312294-acae169d3c8bb536.gif?imageMogr2/auto-orient/strip)

### 1.2、主要功能如下
- 1：单张图片的时候支持按照图片宽高比列在设定区域内自适应
- 2：Adapter方式绑定数据和UI
- 3：图片点击事件回调
- 4：设置图片间隔大小
- 5：自由通过Glide设置ImageView圆角效果

## 二、使用
### 2.1、自定义属性如下
```xml
<resources>
    <declare-styleable name="NineImageLayout">
        <!-- 控件宽高 -->
        <attr name="nine_layoutWidth" format="dimension"/>
        <!-- 单张图片时的最大宽高范围-->
        <attr name="nine_singleImageWidth" format="dimension" />
        <!-- 图片之间间隙大小 -->
        <attr name="nine_imageGap" format="dimension" />
    </declare-styleable>
</resources>
```  

### 2.2、布局中使用自定义NineImageLayout
```xml
 <com.cyq.customview.nineLayout.view.NineImageLayout
        android:id="@+id/nine_image_layout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@+id/tv_title"
        android:layout_marginTop="20dp"
        app:nine_imageGap="4dp"
        app:nine_layoutWidth="300dp"
        app:nine_singleImageWidth="180dp" />
```

### 2.3、Adapter方式绑定数据和UI
其中Glide.asBitmap是为了计算图片宽高，如果后台有返回图片的宽高可以省略这一步，直接setSingleImage(width, height,imageView)，
Ps:如果可以建议后台返回图片宽高，这样可以避免单张图片的时候控件高度跳屏，比如我限制单张图片宽高在200dp范围，要展示宽1000px高500px的时候，在图片未加载完成时控件宽高为200dp，图片加载完成后高度变为100dp，会有一个不好的用户体验，所以建议上传图片的时候记录图片宽高信息
```java
nineImageLayout.setAdapter(new NineImageAdapter() {
            @Override
            protected int getItemCount() {
                return mData.size();
            }

            @Override
            protected View createView(LayoutInflater inflater, ViewGroup parent, int i) {
                return inflater.inflate(R.layout.item_img_layout, parent, false);
            }

            @Override
            protected void bindView(View view, final int i) {
                final ImageView imageView = view.findViewById(R.id.iv_img);
                Glide.with(mContext).load(mData.get(i)).into(imageView);
                if (mData.size() == 1) {
                    Glide.with(mContext)
                            .asBitmap()
                            .load(mData.get(0))
                            .into(new SimpleTarget<Bitmap>() {
                                @Override
                                public void onResourceReady(Bitmap bitmap, Transition<? super Bitmap> transition) {
                                    final int width = bitmap.getWidth();
                                    final int height = bitmap.getHeight();
                                    nineImageLayout.setSingleImage(width, height,imageView);
                                }
                            });
                    Glide.with(mContext).load(mData.get(0)).into(imageView);
                } else {
                    Glide.with(mContext).load(mData.get(i)).into(imageView);
                }
            }

            @Override
            public void OnItemClick(int i, View view) {
                super.OnItemClick(position, view);
                Toast.makeText(mContext, "position:" + mData.get(i), Toast.LENGTH_SHORT).show();
            }
        });
```

###  2.4、列表里面使用
页面放一个RecyclerView
```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".nineLayout.NineImageLayoutActivity">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerview"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />
</FrameLayout>
```
item布局如下
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="20dp">

    <TextView
        android:id="@+id/tv_title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="标题"
        android:textColor="@android:color/black"
        android:textSize="18sp" />

    <com.cyq.customview.nineLayout.view.NineImageLayout
        android:id="@+id/nine_image_layout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@+id/tv_title"
        android:layout_marginTop="20dp"
        app:nine_imageGap="4dp"
        app:nine_layoutWidth="300dp"
        app:nine_singleImageWidth="180dp" />
</RelativeLayout>
```

Activity中构造一下测试数据，大致代码如下
```java
public class NineImageLayoutActivity extends AppCompatActivity {
    private RecyclerView mRecyclerView;
    private MyAdapter mAdapter;
    private Random random;
    private final String URL_IMG = "http://q3x62hkt1.bkt.clouddn.com/banner/58f57dfa5bb73.jpg";
    private final String URL_IMG_2 = "http://q3x62hkt1.bkt.clouddn.com/timg.jpeg";
    private List<List<String>> mList = new ArrayList<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_nine_image_layout);
        random = new Random();
        List<String> testList = new ArrayList<>();
        testList.add(URL_IMG_2);
        for (int i = 0; i < 100; i++) {
            int count = i % 9 + 1;
            List<String> list = new ArrayList<>();
            for (int j = 0; j < count; j++) {
                list.add(URL_IMG);
            }
            if (i % 8 == 0) {
                mList.add(testList);
            }
            mList.add(list);
        }
        mRecyclerView = findViewById(R.id.recyclerview);
        mAdapter = new MyAdapter(mList, this);
        mRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        mRecyclerView.setAdapter(mAdapter);
    }
}
```

MyAdapter中设置数据
```java
import java.util.List;

/**
 * @author : ChenYangQi
 * date   : 2020/1/16 13:49
 * desc   :
 */
public class MyAdapter extends RecyclerView.Adapter<MyAdapter.MyViewHolder> {
    private List<List<String>> mList;
    private Context mContext;

    public MyAdapter(List<List<String>> mList, Context mContext) {
        this.mList = mList;
        this.mContext = mContext;
    }

    @NonNull
    @Override
    public MyViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(mContext).inflate(R.layout.item_nine_img_layout_list, parent, false);
        return new MyViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull final MyViewHolder holder, final int position) {
        final List<String> mData = mList.get(position);
        holder.tvTitle.setText("这是" + mData.size() + "张图片的标题");
        final NineImageLayout nineImageLayout = holder.nineImageLayout;
        holder.nineImageLayout.setAdapter(new NineImageAdapter() {
            @Override
            protected int getItemCount() {
                return mData.size();
            }

            @Override
            protected View createView(LayoutInflater inflater, ViewGroup parent, int i) {
                return inflater.inflate(R.layout.item_img_layout, parent, false);
            }

            @Override
            protected void bindView(View view, final int i) {
                final ImageView imageView = view.findViewById(R.id.iv_img);
                Glide.with(mContext).load(mData.get(i)).into(imageView);
                if (mData.size() == 1) {
                    Glide.with(mContext)
                            .asBitmap()
                            .load(mData.get(0))
                            .into(new SimpleTarget<Bitmap>() {
                                @Override
                                public void onResourceReady(Bitmap bitmap, Transition<? super Bitmap> transition) {
                                    final int width = bitmap.getWidth();
                                    final int height = bitmap.getHeight();
                                    nineImageLayout.setSingleImage(width, height,imageView);
                                }
                            });
                    Glide.with(mContext).load(mData.get(0)).into(imageView);
                } else {
                    Glide.with(mContext).load(mData.get(i)).into(imageView);
                }
            }

            @Override
            public void OnItemClick(int i, View view) {
                super.OnItemClick(position, view);
                Toast.makeText(mContext, "position:" + mData.get(i), Toast.LENGTH_SHORT).show();
            }
        });
    }

    @Override
    public int getItemCount() {
        return mList.size();
    }
 
    class MyViewHolder extends RecyclerView.ViewHolder {
        TextView tvTitle;
        NineImageLayout nineImageLayout;

        public MyViewHolder(@NonNull View itemView) {
            super(itemView);
            tvTitle = itemView.findViewById(R.id.tv_title);
            nineImageLayout = itemView.findViewById(R.id.nine_image_layout);
        }
    }
}

```

## 三、源码地址
具体自定义NineImageLayout过程，可以查看[NineImageLayout](https://github.com/DaLeiGe/AndroidSamples/blob/master/CustomViewDemo/app/src/main/java/com/cyq/customview/nineLayout/view/NineImageLayout.java)
