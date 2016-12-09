
###2016-12-06会议精神

> #####目标

######建立商品相关中间表并创建缓存，优化促销场次，促销专场，品类，分类商品查询性能，优化商品名及关键词搜索性能，根据业务需求方便隐藏显示相关模块（促销场次，专场，分类，品类等）

> #####业务逻辑整理

![](../imgs/lp.png)

* 1.app端动态隐藏显示模块
	* 1.促销场次，促销专场
		> 技术实现

		```markdown
			1. 创建两个redis缓存，类型为hash，结构为community_id:秒团场次/促销专场信息（促销id，
			   开始时间，结束时间）集合，用于显示隐藏秒团场次/促销专场（redis缓存中园区对应场
			   次/专场列表存在即显示）。
			2. 秒团场次在场次审核通过时更新缓存，促销专场在促销专题审核通过时更新缓存，缓存更新查询语
			   句如下：

			   秒团场次：
					SELECT DISTINCT
						dg.id promotionId,
						dg.activity_begintime,
						dg.activity_endtime,
						css.community_id
					FROM
						sys_discount_group dg
					INNER JOIN sys_discount_topic_group dtp ON dtp.discount_group_id = dg.id
					INNER JOIN sys_discount_topic dt ON dt.id = dtp.discount_topic_id
					INNER JOIN sys_discount_product dp ON dp.discount_group_id = dg.id
					INNER JOIN sys_store_product sp ON sp.id = dp.product_id
					INNER JOIN sys_store ss ON ss.id = sp.store_id
					INNER JOIN community_store_subject css ON css.store_id = ss.id
					WHERE
						dt.approve_status = 1
					AND dg.type = 1
					AND dt. STATUS <> 3
					GROUP BY
						css.community_id
					ORDER BY
						dp.activity_begintime DESC

			   促销专场：
					SELECT DISTINCT
						dt.id promotionId,
						dt.activity_begintime,
						dt.activity_endtime,
						css.community_id
					FROM
						sys_discount_group dg
					INNER JOIN sys_discount_topic_group dtp ON dtp.discount_group_id = dg.id
					INNER JOIN sys_discount_topic dt ON dt.id = dtp.discount_topic_id
					INNER JOIN sys_discount_product dp ON dp.discount_group_id = dg.id
					INNER JOIN sys_store_product sp ON sp.id = dp.product_id
					INNER JOIN sys_store ss ON ss.id = sp.store_id
					INNER JOIN community_store_subject css ON css.store_id = ss.id
					WHERE
						dt.approve_status = 1
					AND dg.type = 2
					AND dt.status <> 3
					ORDER BY
						dp.activity_begintime DESC

			   将结果列表根据园区id分组后放入redis缓存中，更新方式为直接删除插入，查询时如果缓
			   存已被清除，将根据上述逻辑重新生成一份，更新到缓存中。
			3. 秒团场次查询方式为遍历缓存中的促销信息里列表，判断结束时间超过当前时间的数据下标，
			   取后面24条数据。专题需要匹配当前时间在开始时间和结束时间之间的数据里列表。
		```
	* 2.分类，品类
		> 技术实现(分类，品类实现方式一致)

		```markdown
			基础思路：创建园区商品关联表并同时创建分类/品类redis缓存（redis缓存类型为set，将分类，
					 品类信息转为json格式存入缓存中，缓存过期时间为15分钟），表内存储商品即为关
					 联园区可显示商品，通过特定操作触发异步更新。最终app调用接口时根据缓存中对
					 应园区是否有分类/品类来显示或隐藏。

			更新触发点：
					1.商家商品：商家商品名称修改，下架(下架，批量下架按钮， 在编辑页面选择下架)  
					2.门店商品：新增，管理后台上架，管理后台批量上架， 管理后台下架，管理后台批
					  量下架，商家助手商品上架，商家助手商品下架
					3.服务分类：分类的关闭，开启
					4.品类：不用管，因为品类下有商品时，不能删除
					5.园区服务：园区服务的开启，关闭， 修改服务门店
					6.公共服务：关闭
					7.门店：门店的开启，关闭
					8.商家：商家的关闭

			sql语句整理：

			创建园区商品表：
				CREATE TABLE sys_community_product (
					community_id VARCHAR (40) NOT NULL COMMENT '园区id',
					product_id VARCHAR (40) NOT NULL COMMENT '门店商品id',
					product_name VARCHAR (200) NOT NULL COMMENT '商品名称',
					section_id VARCHAR (40) DEFAULT '' COMMENT 
											'服务分类id，sys_subject_section.id',
					section_category_id VARCHAR (40) DEFAULT '' COMMENT '服务品类id',
					sub_community_id VARCHAR (40) NOT NULL COMMENT '园区服务id',
					store_id VARCHAR (40) NOT NULL COMMENT '门店id',
					update_time datetime COMMENT '更新时间'
				) ENGINE = INNODB DEFAULT charset = utf8 COMMENT '园区商品表';

				ALTER TABLE sys_community_product ADD CONSTRAINT 
					pk_sys_community_product PRIMARY KEY (community_id, product_id);

				CREATE INDEX idx_sys_community_product_id ON 
					sys_community_product (product_id);

			分类查询：
				SELECT DISTINCT
					ss.id,
					ss.section_name,
					ss.section_img
				FROM
					community_store_subject s
				INNER JOIN sys_store_product p ON s.store_id = p.store_id
				AND p.product_status = 0
				INNER JOIN sys_shop_goods g ON p.goods_id = g.id
				AND g.product_status = 0
				INNER JOIN sys_subject_section ss ON g.section_id = ss.id
				WHERE
					s.community_id = ''
				AND s.sub_status = 0
				AND ss.visible = 1

			商家商品名称修改
				UPDATE sys_community_product
				SET product_name = ''
				WHERE
					product_id IN ('', '', '');
			
			商家商品下架：
				DELETE
				FROM
					sys_community_product
				WHERE
					product_id IN ('', '', '');

			门店商品新增，或者上架:
				DELETE
				FROM
					sys_community_product
				WHERE
					product_id = '';

				INSERT INTO sys_community_product (xxx, xxx, xxx) SELECT
					s.community_id,
					p.id,
					g.product_name,
					g.section_id,
					g.section_category_id,
					s.sub_community_id,
					s.store_id,
					now()
				FROM
					community_store_subject s
				INNER JOIN sys_store_product p ON s.store_id = p.store_id
				AND p.product_status = 0
				INNER JOIN sys_shop_goods g ON p.goods_id = g.id
				AND g.product_status = 0
				INNER JOIN sys_subject_section ss ON g.section_id = ss.id
				WHERE
					s.sub_status = 0
				AND ss.visible = 1
				AND p.id = '';
			    
			门店商品的下架：
				DELETE
				FROM
					sys_community_product
				WHERE
					product_id = '';

			服务分类开启：
				DELETE
				FROM
					sys_community_product
				WHERE
					section_id = '';

				INSERT INTO sys_community_product (xxx, xxx, xxx) SELECT
					s.community_id,
					p.id,
					g.product_name,
					g.section_id,
					g.section_category_id,
					s.sub_community_id,
					s.store_id,
					now()
				FROM
					community_store_subject s
				INNER JOIN sys_store_product p ON s.store_id = p.store_id
				AND p.product_status = 0
				INNER JOIN sys_shop_goods g ON p.goods_id = g.id
				AND g.product_status = 0
				INNER JOIN sys_subject_section ss ON g.section_id = ss.id
				WHERE
					s.sub_status = 0
				AND ss.visible = 1
				AND ss.id = '';
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

![](../imgs/zb.png)

