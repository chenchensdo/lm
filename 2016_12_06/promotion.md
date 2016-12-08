
###2016-12-06会议精神

> #####目标

######建立商品相关中间表并创建缓存，优化促销场次，促销专场，品类，分类商品查询性能，优化商品名及关键词搜索性能，根据业务需求方便隐藏显示相关模块（促销场次，专场，分类，品类等）

> #####业务逻辑整理

* 1.app端动态隐藏显示模块
	* 1.促销场次，促销专场
		> 技术实现

		```markdown
			1. 创建商品园区促销商品表sys_community_promotion_product(暂定)，字段
			   包含园区id，促销场次/专场id，类型（场次或专场），商品id，促销开始时间，促销结束时
			   间。园区id，促销id作为唯一索引。
			2. 创建redis缓存，类型为hash，结构为community_id:促销场次/专场id集合，用于显示隐藏促
			   销场次/专场（redis缓存中园区对应场次/战场列表存在即显示）。
			3. 创建定时任务，一小时执行一次，查询状态正常的促销商品列表，语句如下：
					SELECT
						dg.id promotionId,
						dg.type,
						dp.id productId,
						dg.activity_begintime,
						dg.activity_endtime,
						css.community_id
					FROM
						sys_discount_group dg
					INNER JOIN sys_discount_product dp ON dp.discount_group_id = dg.id
					AND life_status = 0
					INNER JOIN sys_store_product sp ON sp.id = dp.product_id
					INNER JOIN sys_store ss ON ss.id = sp.store_id
					INNER JOIN community_store_subject css ON css.store_id = ss.id
					WHERE
						dg.approve_status = 1
					AND dg.stauts <> 3
			   将结果列表插入sys_community_promotion_product，更新方式为直接删除插入，执行完成后，
			   查询园区对应场次专场列表去重放入redis缓存中。
		```
	* 2.分类，品类
		> 技术实现(分类，品类实现方式一致)

		```markdown
			基础思路：创建园区商品关联表并同时创建分类/品类redis缓存（redis缓存类型为hash，结构
					 为communityId:可显示的分类/品类id集合，园区商品表更新后同步更新缓存，如缓存
					 被不小心清除，可实时查询园区商品表并同步到缓存中），表内存储商品即为关联园
					 区可显示商品，定时更新表数据（商品下架，关闭，服务关闭等状态延时清除相关园
					 区商品）。门店商品上架，批量上架，新增等操作时实时更新园区商品表。最终app调
					 用接口时根据缓存中对应园区是否有分类/品类来显示或隐藏。

			实现步骤：
			1. 创建商品园区关联表，sys_community_product(暂定),字段包含园区id，商品id，
			   分类id，品类id，园区服务id。
			2. 定时更新表数据及redis缓存，更新逻辑为
			   (1. 建立一个园区更新时间表sys_community_update_time(暂定)，包含园区，更新时间两个
			   	字段，用于存储园区数据更新时间，首次执行会将所有园区插入此表，更新时间为当前时间
			   	减一天，另起一个定时任务每10分钟检查是否存在新增园区，有新增园区则插入。
			   (2. 每次从园区更新时间表中查询一个更新时间超过当前时间15分钟的园区，
			       通过以下语句：
				   	SELECT
							p.id,
							g.product_name,
							g.section_id,
							g.section_category_id,
							msub.sub_community_id
						FROM
							sys_store_product p
						INNER JOIN sys_shop_goods g ON p.goods_id = g.id
						INNER JOIN community_store_subject msub ON g.sub_id = msub.sub_id
						AND msub.community_id = ''
						WHERE
							msub.sub_status = 0
						AND g.product_status = 0
						AND p.product_status = 0
				   查询该园区的所有商品信息，直接插入sys_community_product表中，更新方式为删除重
				   新插入。
			   (3. 更新品类缓存，根据园区商品表查询园区对应品类去重，将结果更新至品类缓存中。
			   (4. 更新分类缓存，分类缓存除了第三步操作，还需添加一个操作，查询园区服务中服务状态
			   	为正常的h5服务所属分类列表根据分类去重。将该列表中分类也加入到缓存中所属园区的对
			   	应分类列表中。
			3. 门店商品上线，批量上线，新增时实时更新园区商品表并更新缓存信息，详细逻辑与第二步基
			   本一致，不同在于需要更新商品所属门店提供服务的所有园区。
		```

* 2.商品名，关键词搜索
	* 1.创建搜索关键词表sys_keyword_hot(暂定)，字段包含关键词，搜索次数，关联商品id，园区id,首次
		搜索时间，最后搜索时间。搜索时更新此表。
	* 2.商品名搜索改为园区商品表。

* 3.商圈首页，热门商品排序逻辑整理
	* 1.将热门商品，普通商品查询分为两个接口，创建一个用于存储热门商品的redis，类型为hash，
		结构为community_id:热门商品id集合，热门商品根据商家产品修改时间倒序查询最多10个。将
		查询结果放入缓存中。
	* 2.普通商品根据销量倒序，是否促销，商家产品修改时间倒序并
		过滤redis缓存中的热门商品进行查询每页20条数据。


* 4.分类商品显示逻辑整理
	* 1.将分类中新增商品，普通商品查询分为两个接口，新增商品查询根据最近三天添加并按照门店
		商品新增时间倒序，若三天内无新增商品，数据不显示。

	* 2.普通商品根据销量倒序，是否促销，商家产品修改时间倒序并新增时间大于3天进行查询每页
		20条数据，如果新增商品超过20个则不执行查询。

> #####定时任务更新

* 1.园区商品信息更新

* 2.促销场次，促销专场缓存更新

> #####影响点

* 1.下架/失效商品相关分类/品类延时反应

* 2.促销信息延时显示

* 3.商品查询语句调整，分类/品类商品关联园区商品表，促销商品关联园区促销商品表

