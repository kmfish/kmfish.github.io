---
title: MultiTypeListViewAdapter Android ListView 多type的Adapter封装
tags: Android ListView Adapter ViewType
date: 2016-01-16

---

## [MultiTypeListViewAdapter](https://github.com/kmfish/MultiTypeListViewAdapter) 是什么？
MultiTypeListViewAdapter，顾名思义。其封装了多type下的Adapter的编程模式，通过对每种type统一接口，利用多态的方式，将type的实现从Adapter中抽离出来。Adapter只需面向统一接口，所以可以提供一个通用实现，实现代码不再变动。而会变动的每个type对应的item实现，则由使用者自己去实现。对扩展开放，对修改封闭。

同时，由于每个type的item均被抽离出来了。相当于复用的粒度为每个type item，可以根据需要，动态地选择合适的item去添加到adapter中。提高了代码复用，每个人编写维护好自己的item即可，避免了多人合作时都去修改Adapter，容易造成冲突。

另外，由于ViewHolder 模式的规范，MultiTypeListViewAdapter也同时封装了ViewHolder模式。
<!--more-->
---

## 常见的ListView Adapter 实现
先看一下常见的ListAdapter 实现，分为单个type和多type两种情况。

### 1、单Type的Adapter
```java
class ListAdapter extend BaseAdapter {
    ...
    private List<String> contents = new ArrayList();
    ...
    public View getView(int position, View convertView, ViewGroup parent) {
            String item = getItem(position);
            if(null == item) {
                throw new RuntimeException("list item is never null. pos:" + position);
            } else {
                ViewHolder holder;
                if(null == convertView) {
                    holder = createViewHolder(parent);
                    convertView = holder.itemView;
                    convertView.setTag(holder);
                } else {
                    holder = (ViewHolder)convertView.getTag();
                }
   
                item.updateHolder(holder, position);
                return convertView;
            }
        }
    }
   
    private ViewHolder createViewHolder(ViewGroup parent) {
        // create item view from layout xml
        ...
    }
   
    private void updateHolder(ViewHolder holder, String item) {
        // update item view
        holder.text.setText(item);
    }
   
    private class ViewHolder {
        TextView text;
       
        ViewHolder(View itemView) {
            text = (TextView)itemView.findViewById(R.id.text);
        }
    }
    ...
}
```

应该是挺常见的写法吧，继续往下。

### 2、多type的adapter
```java
class ListAdapter extend BaseAdapter {
    ...
    private List<String> contents = new ArrayList();
    ...
    public int getItemViewType(int position) {
        if (position % 2 == 0) {
            return 0;
        }
   
        return 1;
    }

    public int getViewTypeCount() {
        return 2;
    }
   
    public View getView(int position, View convertView, ViewGroup parent) {
            String item = getItem(position);
            if(null == item) {
                throw new RuntimeException("list item is never null. pos:" + position);
            } else {
                ViewHolder holder;
                if(null == convertView) {
                    if (getItemViewType(position) == 0) {
                        holder = createViewHolder(parent);   
                    } else if (getItemViewType(position) == 1) {
                        holder = createViewHolder1(parent);
                    }
                   
                    convertView = holder.itemView;
                    convertView.setTag(holder);
                } else {
                    holder = (ViewHolder)convertView.getTag();
                }
               
                if (getItemViewType(position) == 0) {
                    item.updateHolder(holder, position);
                } else if (getItemViewType(position) == 1) {
                    item.updateHolder1(holder, position);
                }
               
                return convertView;
            }
        }
    }
   
    private ViewHolder createViewHolder(ViewGroup parent) {
        // create item view from layout xml
        ...
    }
   
    private void createViewHolder1(ViewGroup parent) {
        // create item view from layout xml
        ...
    }
   
    private void updateHolder(ViewHolder holder, String item) {
        // update item view
        holder.text.setText(item);
    }
   
    private void updateHolder1(ViewHolder1 holder, String item) {
        // update item view
        holder.text.setText(item);
        holder.img.setImageResoure(R.drawable.img);
    }
   
    private class ViewHolder {
        TextView text;
       
        ViewHolder(View itemView) {
            text = (TextView)itemView.findViewById(R.id.text);
        }
    }
   
    private class ViewHolder1 {
        TextView text;
        ImageView img;
       
        ViewHolder(View itemView) {
            text = (TextView)itemView.findViewById(R.id.text);
            img = (ImageView)itemView.findViewById(R.id.img);
        }
    }
    ...
}
```
看到这里，是否感觉这个Adapter的getView（）开始有些让人不舒服了呢。若type再进一步增加，难不成还得继续if/else下去，adapter变得又臭又长。估计往后下去，就再没人愿意维护了。而且，这个Adapter的这一堆代码还得到处重复写下去，每个有listView的地方，都得配套一个adapter。针对这个坑，我设计了MultiTypeListViewAdapter来解决。希望能帮助到有需要的程序猿们~


---

## 使用 MultiTypeListViewAdapter
```java
public class MainActivity extends AppCompatActivity {

    private ListView listView;
    private MultiTypeArrayAdapter adapter;

    private static final int ITEM_TYPE_1 = 0;
    private static final int ITEM_TYPE_2 = 1;

    private static final int ITEM_TYPE_COUNT = 2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        listView = (ListView) findViewById(R.id.listview);
        adapter = new MultiTypeArrayAdapter(ITEM_TYPE_COUNT);
        listView.setAdapter(adapter);

        setupAdapter();
    }

    private void setupAdapter() {
        adapter.setTypeCount(ITEM_TYPE_COUNT);
        LineListItem1 item1 = new LineListItem1(this, ITEM_TYPE_1, "line type 1");
        LineListItem2 item2 = new LineListItem2(this, ITEM_TYPE_2, "line type 2");

        adapter.setNotifyOnChange(false);
        for (int i = 0, len = 50; i < len; i++) {
            adapter.addItem( i  % 2 == 0 ? item1 : item2);
        }
        adapter.notifyDataSetChanged();
    }

}

public class LineListItem1 extends BaseListItem {

    private String line;

    public LineListItem1(Context mContext, int viewType, String line) {
        super(mContext, viewType);
        this.line = line;
    }

    @Override
    protected int onGetItemLayoutRes() {
        return R.layout.list_item1;
    }

    @Override
    protected ViewHolder onCreateViewHolder(View itemView) {
        return new LineViewHolder(itemView);
    }

    @Override
    public void updateHolder(ViewHolder holder, int pos) {
        LineViewHolder h = (LineViewHolder) holder;
        h.text.setText(line + "_" + pos);
    }

    private class LineViewHolder extends ViewHolder {

        TextView text;

        public LineViewHolder(View itemView) {
            super(itemView);
            text = (TextView) itemView.findViewById(R.id.line_text);
        }
    }
}

public class LineListItem2 extends BaseListItem {

    private String line;

    public LineListItem2(Context mContext, int viewType, String line) {
        super(mContext, viewType);
        this.line = line;
    }

    @Override
    protected int onGetItemLayoutRes() {
        return R.layout.list_item2;
    }

    @Override
    protected ViewHolder onCreateViewHolder(View itemView) {
        return new LineViewHolder2(itemView);
    }

    @Override
    public void updateHolder(ViewHolder holder, int pos) {
        LineViewHolder2 h = (LineViewHolder2) holder;
        h.text.setText(line + "_" + pos);
        h.img.setImageResource(R.drawable.icon_git);

    }

    private class LineViewHolder2 extends ViewHolder {
        ImageView img;
        TextView text;

        public LineViewHolder2(View itemView) {
            super(itemView);
            img = (ImageView) itemView.findViewById(R.id.thumb);
            text = (TextView) itemView.findViewById(R.id.name);
        }
    }
}
```

可以看到，使用MultiTypeListViewAdapter之后，实现多type的Adapter变得相当简单了。再也不用面对一堆判断itemType的if/else了。每增加一种type，只需增加一种新的ListItem即可，再也不用去改动Adapter的代码了。

项目发布在github上了，[MultiTypeListViewAdapter](https://github.com/kmfish/MultiTypeListViewAdapter)，欢迎fork和交流。




