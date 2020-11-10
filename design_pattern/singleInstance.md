# 单例模式

- 单例模式模式可以减少初始化的消耗
- 有人说单例模式其实没啥用，除了在写法上有了编辑器的加持，非常简便实用（好写），这特么还不够吗。
- 他们的理由是：在逻辑层，或者是数据层，很少有情况会调用多次
- 这一次，举个例子，其实并不少用哦。

```

- controller

public function payList(Request $request)
{
    if ($request->ajax()) {
        $result_data = $this->rechargeOrderModel->getTableData($request);
        return api($result_data);
    }
    return view('admin.datamonitor.pay_list', [
        'exporess'      => true,
        "platforms"     => $request["__platforms"] ?? collect([]),
    ]);
}



- model 

public static $withoutAppends = true;
protected $appends = ['status_cn','platform_cn','product_cn'];


public function getStatusCnAttribute()
{
    if(self::$withoutAppends){
        return '';
    }
    return $this->attributes["status"] == 1 ? "已处理" : "未处理";
}


public function getPlatformCnAttribute()
{
    if(self::$withoutAppends){
        return '';
    }
    $platform_data = Platform::all()->pluck("platform_name", "platform_id");
    return isset($platform_data[$this->attributes["platform_idx"]]) ? $platform_data[$this->attributes["platform_idx"]] : "未定义:" . $this->attributes["platform_idx"];
}


public function getProductCnAttribute()
{
    if(self::$withoutAppends){
        return '';
    }
    $recharge_goods_data = RechargeGoodsModel::all()->pluck("name", "commodity_id");
    return isset($recharge_goods_data[$this->attributes["product_id"]]) ? $recharge_goods_data[$this->attributes["product_id"]] : "未知商品";
}



/**
    * 列表查询
    * @param $platform
    * @param $request
    * @return array
    */
public function getTableData($request)
{


    $model = $this->newQuery();
    //兼容js 格式化时间
    if(!empty($request["s_date"]) && !empty($request["e_date"])){
        $request["date"]=$request["s_date"]." 至 ".$request["e_date"];
    }
    $date=formatBetweenDate($request,"datetime");

    $model->when($request["platform_ids"],function ($query,$platform_ids){
        return $query->whereIn("{$this->table}.platform_idx",explode(",",$platform_ids));

    })->when($request["game_id"],function ($query,$account_order){
        return $query->whereRaw("concat(`game_id`,`order_id`) like '%" . $account_order . "%'");

    })->when($date,function ($query,$date_berween){
        return $query->whereBetween("create_time",$date_berween);

    })->when($request["goods_name"],function ($query,$goods_name){
        $product_ids = RechargeGoodsModel::where('name','like',"%{$goods_name}%")->pluck('commodity_id');
        return $query->whereIn("product_id",$product_ids);
    });

    //平台检测
    if ($request["is_admin"] === false) {
        $model = $this->_servers_platforms($model, $request, "{$this->table}", "platform_idx");
    }

    //数据库都使用别名 防止命名错误
    $dbconfigs   = config("database.connections.game_db");
    $db_raw=$dbconfigs['fish_center_account']['database'] . '.' . $dbconfigs['fish_center_account']['prefix'] .
        "{$this->table_account} as " . $dbconfigs['fish_center_log']['prefix'] . "{$this->table_account}";

    self::$withoutAppends = false;
    $model = $model->from($this->table, "recharge")
                        ->select("{$this->table_account}.game_id", "{$this->table}.*")
                        ->leftJoin(DB::raw($db_raw),
                            function ($join) {
                                $join->on(
                                    "{$this->table}.account_idx",
                                    '=',
                                    "{$this->table_account}.account_idx"
                                );
                            })
                        ->orderBy("create_time", "desc");

    //导出
    if($request->input('export',0)){
        return $model->get()->toArray();
    }
    $result_data = $model->paginate($request->input('limit') ?? config("enum.PAGE_NUM"));
    return $result_data;
}


```



- 利用 lavavel 的 append 属性，自动转化字段
- 啃爹的是，有些字段的转化，需要取用数据库，于是就循环取用了
- sql 日志如下

```

2020-11-10 20:07:53: select * from `ga_admins` where `id` = 1 and `ga_admins`.`deleted_at` is null limit 1
2020-11-10 20:07:53: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:53: select * from `ga_permissions` where `name` = 'admin.datamonitor.paylist' limit 1
2020-11-10 20:07:53: insert into `ga_admin_logs` (`admin_id`, `name`, `operation_model`, `url`, `operation_pram`, `created_at`, `ip`, `mark`) values (1, '超级管理员', 'GET', 'admin.datamonitor.paylist', '[]', '2020-11-10 20:07:53', '127.0.0.1', '充值数据')
2020-11-10 20:07:53: select count(*) as aggregate from `log_recharge` as `log_recharge` left join fish3_center_account_1.account as log_account on `log_recharge`.`account_idx` = `log_account`.`account_idx`
2020-11-10 20:07:53: select `log_account`.`game_id`, `log_recharge`.* from `log_recharge` as `log_recharge` left join fish3_center_account_1.account as log_account on `log_recharge`.`account_idx` = `log_account`.`account_idx` order by `create_time` desc limit 10 offset 0
2020-11-10 20:07:53: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:53: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`
2020-11-10 20:07:54: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:07:54: select * from `ga_recharge_goods`


```

- 你没有看错，一页十条数据，ga_recharge_goods 和 ga_platform 这两张表，分别查询了十次






## 单例模式的模式的优化

- 一个单例模式是对象的调用采用单例模式（Model 调用静态方法，和单例调用普通方法，这两者的优劣，暂时不知）
- 一个单例模式是将查询到的结果保存在静态变量或者常规的变量中，这是最大的优化之处

```

    private static $staticRechargeGoods = []; ## 静态变量保存数据，减少查询
    private static $staticPlatform = []; ## 静态变量保存数据，减少查询

    /**
     * @return array|\Illuminate\Support\Collection
     * 获取商品信息，一个进程只获取一次
     */
    public function getRechargeGoodsArrCommodity()
    {
        if (!self::$staticRechargeGoods) {
            self::$staticRechargeGoods = RechargeGoodsModel::all()->pluck("name", "commodity_id");
        }
        return self::$staticRechargeGoods;
    }

    /**
     * @return array|\Illuminate\Support\Collection
     * 获取平台信息，一个进程至获取一次
     */
    public function getPlatformArr()
    {
        if (!self::$staticPlatform) {
            self::$staticPlatform = Platform::all()->pluck("platform_name", "platform_id");
        }
        return self::$staticPlatform;
    }


    public function getPlatformCnAttribute()
    {
        if(self::$withoutAppends){
            return '';
        }
        $platform_data = self::getInstance()->getPlatformArr();
        return isset($platform_data[$this->attributes["platform_idx"]]) ? $platform_data[$this->attributes["platform_idx"]] : "未定义:" . $this->attributes["platform_idx"];
    }


    public function getProductCnAttribute()
    {
        if(self::$withoutAppends){
            return '';
        }
        $recharge_goods_data = self::getInstance()->getRechargeGoodsArrCommodity();
        return isset($recharge_goods_data[$this->attributes["product_id"]]) ? $recharge_goods_data[$this->attributes["product_id"]] : "未知商品";
    }



```




- sql 日志如下：

```

2020-11-10 20:29:13: select * from `ga_admins` where `id` = 1 and `ga_admins`.`deleted_at` is null limit 1
2020-11-10 20:29:13: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:29:13: select * from `ga_permissions` where `name` = 'admin.datamonitor.paylist' limit 1
2020-11-10 20:29:13: insert into `ga_admin_logs` (`admin_id`, `name`, `operation_model`, `url`, `operation_pram`, `created_at`, `ip`, `mark`) values (1, '超级管理员', 'GET', 'admin.datamonitor.paylist', '[]', '2020-11-10 20:29:13', '127.0.0.1', '充值数据')
2020-11-10 20:29:13: select count(*) as aggregate from `log_recharge` as `log_recharge` left join fish3_center_account_1.account as log_account on `log_recharge`.`account_idx` = `log_account`.`account_idx`
2020-11-10 20:29:13: select `log_account`.`game_id`, `log_recharge`.* from `log_recharge` as `log_recharge` left join fish3_center_account_1.account as log_account on `log_recharge`.`account_idx` = `log_account`.`account_idx` order by `create_time` desc limit 10 offset 0
2020-11-10 20:29:13: select * from `ga_platform` where `ga_platform`.`deleted_at` is null
2020-11-10 20:29:13: select * from `ga_recharge_goods`




```


- 从日志可以看出，ga_recharge_goods 和 ga_platform 这两张表只查询了一次


