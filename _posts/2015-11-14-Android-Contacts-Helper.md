---
layout: post
title: 通讯录查询和添加
subtitle:   "Android 通讯录的使用总结，可以帮助你少走弯路"
date:       2015-11-14
author:     "Robert Zhang"
header-img: "img/post-bg-android.jpg"
tags:
    - Android
---

# 通讯录小记
本文概要：
在获取通讯录的时候会遇到各种坑。虽然网上也有不少关于该部分的内容，但大多数不能满足我的需求。所以按照一贯的风格，自己动手丰衣足食。

### 通讯录中遇到的坑
通讯录是我们每天都会用到的应用，算是我们再也熟悉不过的。最近一段时间一直在做关于电话的应用，或多或少的会获取通讯录信息。这部分代码的分析网上有一大堆。感兴趣的同学可以自己搜索学习。但在我看来这部分的内容，其实还是有点复杂的。获取通讯录的时候会牵扯到很多张表，刚开始的时候确实会让人摸不着脉。特别是网上的资料，因为作者出发的角度不同，所以使用的API也会不一样。这样会给初学者带来很多烦恼，从而使他们无法清楚的知道该如何正确的定制属于他们的需要。

### 填坑
通讯录的使用无非就是查询、添加和删除。是的，你没有听错就是万能的增删改查。在这里我对查询和添加做了一个封装，在大多数情况下基本能够满足我们的需要了。

#### 1、查询
查询的核心代码如下。

``` java
// 本地通讯录数据
	public List<Contacts> contacts; //这里的Contacts是我们自定义的类
	
//加载本地通讯录
	public void loadContacts() {
		contacts = new ArrayList<Contacts>();
		new AsyncTask<Object, Object, String>(){
			protected String doInBackground(Object... arg0) {
				String id;
				String mimetype;
				ContentResolver contentResolver = BaseApplication.getContextObject().getContentResolver();
				//只需要从Contacts中获取ID，其他的都可以不要，通过查看上面编译后的SQL语句，可以看出将第二个参数
				//设置成null，默认返回的列非常多，是一种资源浪费。
				Cursor cursor = contentResolver.query(android.provider.ContactsContract.Contacts.CONTENT_URI,
						new String[]{android.provider.ContactsContract.Contacts._ID}, null, null, null);
				Contacts contactitem;
				while(cursor.moveToNext()) {
					contactitem = new Contacts();//查询的要创建新的contacts对象
					id=cursor.getString(cursor.getColumnIndex(android.provider.ContactsContract.Contacts._ID));
					//从一个Cursor获取所有的信息
					Cursor contactInfoCursor = contentResolver.query(
							android.provider.ContactsContract.Data.CONTENT_URI,
							new String[]{android.provider.ContactsContract.Data.CONTACT_ID,
									android.provider.ContactsContract.Data.MIMETYPE,
									android.provider.ContactsContract.CommonDataKinds.StructuredName.GIVEN_NAME, //用户名
									android.provider.ContactsContract.CommonDataKinds.Organization.COMPANY,//公司
									android.provider.ContactsContract.CommonDataKinds.Organization.TITLE,//职称
									android.provider.ContactsContract.CommonDataKinds.Phone.TYPE,//电话属性
									android.provider.ContactsContract.CommonDataKinds.Phone.NUMBER,// 电话号码
									android.provider.ContactsContract.CommonDataKinds.Email.TYPE,//邮件属性
									android.provider.ContactsContract.CommonDataKinds.Email.DATA,//邮件信息
									android.provider.ContactsContract.CommonDataKinds.Note.NOTE,//备注
									android.provider.ContactsContract.Data.DATA1,//查询的细节
									android.provider.ContactsContract.Data.DATA5
							}, 
							android.provider.ContactsContract.Data.CONTACT_ID+"="+id, null, null);
					while(contactInfoCursor.moveToNext()) {
						mimetype = contactInfoCursor.getString(
								contactInfoCursor.getColumnIndex(android.provider.ContactsContract.Data.MIMETYPE));

						if (mimetype.equals(StructuredName.CONTENT_ITEM_TYPE)) {
							String name = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.StructuredName.GIVEN_NAME));
							//LogUtils.E("====name==="+name);
							contactitem.setName(name);
						} else if (mimetype.equals(Organization.CONTENT_ITEM_TYPE)) {
							String org = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Organization.COMPANY));
							String title = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Organization.TITLE));
							String department = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Organization.DEPARTMENT));
							//LogUtils.E("====org==="+org+","+title);
							contactitem.setOrganization(org);
							contactitem.setJobtitle(title);
							contactitem.setDepartment(department);
						} else if (mimetype.equals(Phone.CONTENT_ITEM_TYPE)) {
							String type = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Phone.TYPE));
							String number;
							String phonename = null;
							int a = Integer.parseInt(type);
							switch (a){
							case Phone.TYPE_MOBILE:
								phonename = "mobile";
								break;
							case Phone.TYPE_MAIN:
								phonename = "main";
								break;
							case Phone.TYPE_HOME:
								phonename = "home";
								break;
							case Phone.TYPE_WORK:
								phonename = "work";
								break;
							case Phone.TYPE_FAX_WORK:
								phonename = "fax_work";
								break;
							case Phone.TYPE_FAX_HOME:
								phonename = "fax_home";
								break;
							case Phone.TYPE_OTHER:
								phonename = "other";
								break;
							case Phone.TYPE_CUSTOM:
								phonename = "custom";
								break;
							}
							number = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Phone.NUMBER));
							//LogUtils.E("====phone==="+phonename+","+number);
							ContactFields cf = new ContactFields("Contact::Phone", phonename, number);
							contactitem.contact_fields.add(cf);

						} else if (mimetype.equals(Email.CONTENT_ITEM_TYPE)) {
							String type = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Email.TYPE));
							String emailname = null;
							switch (Integer.parseInt(type)) {
							case Email.TYPE_WORK:
								emailname = "work";
								break;
							case Email.TYPE_CUSTOM:
								emailname = "custom";
								break;
							case Email.TYPE_MOBILE:
								emailname = "mobile";
								break;
							case Email.TYPE_HOME:
								emailname = "home";
								break;
							case Email.TYPE_OTHER:
								emailname = "other";
								break;
							}
							String date = contactInfoCursor.getString(contactInfoCursor  
									.getColumnIndexOrThrow(android.provider.ContactsContract.CommonDataKinds.Email.DATA));
							//LogUtils.E("====email==="+emailname+","+date);
							ContactFields cf = new ContactFields("Contact::Email", emailname, date);//ContactFields也是我自定义的一个对象，你可以根据需要定义自己的bean类
							contactitem.contact_fields.add(cf);

						} else if (mimetype.equals(Note.CONTENT_ITEM_TYPE)) {
							String note = contactInfoCursor.getString(
									contactInfoCursor.getColumnIndex(android.provider.ContactsContract.CommonDataKinds.Note.NOTE));
//							LogUtils.E("====note==="+note);
							contactitem.setNote(note);
						}
					}
					contacts.add(contactitem);
//					System.out.println("*********");
					contactInfoCursor.close();
				}
				cursor.close();
			
				return null;
			}
			
		}.execute();
	}
```

在上面的代码中mimetype属性是十分重要的。它是android.provider.ContactsContract.Data.MIMETYPE属性的内容。通过它我们可以判断当前获取数据表的列是哪种CONTENT_ITEM_TYPE。之后根据需要在对应的mimetype下，通过android.provider.ContactsContract.CommonDataKind来获取对应索引列下的内容。代码中的关键部分我都做了注释。至于android.provider.ContactsContract的用法和详细说明，请自行查找相关文档，在此不做过多介绍。

#### 2、添加
添加的道理也是一样，根据需要通过mimetype往通讯录数据库中添加内容。内容不是很复杂，直接上代码

``` java
/** 
	 * 添加新的联系人
	 * @param name 名称 -- 不可为空
	 * @param company 公司
	 * @param position 职位
	 * @param numberlist 电话列表
	 * @param emaillist 邮箱列表
	 * @param note 备注
	 */
	public static void addContacts(Bitmap avatar, String name, String company, String position, List<AddContactsNumberItem> numberlist,
			List<AddContactsNumberItem> emaillist, String note){//AddContactsNumberItem为我自定义类，用来存放phone，email这样的存在多条记录的属性值

		ContentValues values = new ContentValues();
		// 首先向RawContacts.CONTENT_URI执行一个空值插入，目的是获取系统返回的rawContactId
		Uri rawContactUri = EPApplication.getContextObject().getContentResolver().insert(
				RawContacts.CONTENT_URI, values);//EPApplication是我定义的Application的子类，getContextObject方法返回的是context
		long rawContactId = ContentUris.parseId(rawContactUri);
		// 表插入姓名数据
		values.clear();
		values.put(Data.RAW_CONTACT_ID, rawContactId);
		values.put(Data.MIMETYPE, StructuredName.CONTENT_ITEM_TYPE);// 内容类型
		values.put(StructuredName.GIVEN_NAME, name);
		EPApplication.getContextObject().getContentResolver().insert(ContactsContract.Data.CONTENT_URI,
				values);

		//添加公司和地址
		if (company != null || position != null) {
			values.clear();
			values.put(Data.RAW_CONTACT_ID, rawContactId);
			values.put(Data.MIMETYPE, Organization.CONTENT_ITEM_TYPE);
			if(company != null){
				values.put(Organization.COMPANY, company);
			}
			if (position != null) {
				values.put(Organization.TITLE, position);
			}
			values.put(Organization.TYPE, Organization.TYPE_WORK);
			EPApplication.getContextObject().getContentResolver().insert(ContactsContract.Data.CONTENT_URI,
					values);
		}
		
		// 插入电话数据
		if (numberlist != null && numberlist.size()>0) {
			for (AddContactsNumberItem item : numberlist) {
				int type = 0;
				if (item.getName().equals("手机")) {
					type = Phone.TYPE_MOBILE;
				} else if (item.getName().equals("工作")){
					type = Phone.TYPE_WORK;
				} else if (item.getName().equals("家庭")){
					type = Phone.TYPE_HOME;
				} else if (item.getName().equals("工作传真")){
					type = Phone.TYPE_FAX_WORK;
				} else if (item.getName().equals("家庭传真")){
					type = Phone.TYPE_FAX_HOME;
				} else if (item.getName().equals("其他")){
					type = Phone.TYPE_OTHER;
				}
				values.clear();
				values.put(Data.RAW_CONTACT_ID, rawContactId);
				values.put(Data.MIMETYPE, Phone.CONTENT_ITEM_TYPE);
				values.put(Phone.NUMBER, item.getContent());
				values.put(Phone.TYPE, type);
				EPApplication.getContextObject().getContentResolver().insert(ContactsContract.Data.CONTENT_URI,
						values);
			}
		}
		
		//添加邮箱
		if (emaillist != null && emaillist.size()>0) {
			for (AddContactsNumberItem item : emaillist) {
				int type = 0;
				if (item.name.equals("工作")) {
					type = Email.TYPE_WORK;
				} else if (item.name.equals("家庭")) {
					type = Email.TYPE_HOME;
				} else if (item.name.equals("其他")) {
					type = Email.TYPE_OTHER;
				}
				values.clear();
				values.put(Data.RAW_CONTACT_ID, rawContactId);
				values.put(Data.MIMETYPE, Email.CONTENT_ITEM_TYPE);
				values.put(Email.DATA, item.getContent());
				values.put(Email.TYPE, type);
				EPApplication.getContextObject().getContentResolver().insert(ContactsContract.Data.CONTENT_URI,
						values);
			}
		}
		
		// 添加备注
		if (note != null && !note.isEmpty()) {
			values.clear();
			values.put(Data.RAW_CONTACT_ID, rawContactId);
			values.put(Data.MIMETYPE, Note.CONTENT_ITEM_TYPE);
			values.put(Note.NOTE, note);
			EPApplication.getContextObject().getContentResolver().insert(ContactsContract.Data.CONTENT_URI,
					values);
		}
		
		// 设置头像
		if (avatar != null) {
			values.clear();
			values.put(Data.RAW_CONTACT_ID, rawContactId);
			values.put(Data.MIMETYPE, Photo.CONTENT_ITEM_TYPE);
			ByteArrayOutputStream array = new ByteArrayOutputStream();
			avatar.compress(Bitmap.CompressFormat.JPEG, 80, array);
			values.put(Photo.PHOTO, array.toByteArray());
			EPApplication.getContextObject().getContentResolver().insert(ContactsContract.Data.CONTENT_URI,
					values);
		}
	}
```

### 小结
每个人对通讯录的理解都有不同，这也说明通讯录使用的灵活性。不管是通讯录还是其他的什么知识，我们都可以用方便自己的形式进行记录和理解。最近一段时间看了很多书，对IOS的开发也在继续深入中。也许正是因为开始了一门新的语言swift，所以才会让我对“吃饭的家伙”Android，有了更深和更客观的认识。如果可能的话，还是多了解一门语言吧。确实会有很多说不出的帮助和提升。也许明年会研究一下HTML5和JS吧。明天的事谁又说的准呢！


