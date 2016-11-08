## 課題 (ルータの CLI を作ろう)

ルータのコマンドラインインタフェース (CLI) を作ろう。

次の操作ができるコマンドを作ろう。

* ルーティングテーブルの表示
* ルーティングテーブルエントリの追加と削除
* ルータのインタフェース一覧の表示
* そのほか、あると便利な機能

コントローラを操作するコマンドの作りかたは、第3回パッチパネルで作った patch_panel コマンドを参考にしてください。


##解答

このレポートは錦織君のレポートを[参考](https://github.com/handai-trema/simple-router-Shu-NISHIKORI/blob/develop/report/report.md)に作成しました。


###ソースコード
* [simple_router.rb](https://github.com/handai-trema/simple-router-Nagatomi-Ken/blob/develop/lib/simple_router.rb)  
* [routing_table.rb](https://github.com/handai-trema/simple-router-Nagatomi-Ken/blob/develop/lib/routing_table.rb)  
* [routing_table](https://github.com/handai-trema/simple-router-Nagatomi-Ken/blob/develop/bin/routing_table)  

###ルーティングテーブルの表示
書き加えたコードを以下に示す。

* routing_table
```
  include Pio
  desc 'show routing table'
  command :show_db do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      puts("routing table")
      db = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
        show_db
      db.each do |each|
        each.each_key do |key|
          print "Dest : ", IPv4Address.new(key).to_s, "/", key.to_i, "\n"
          print "Next : ", each[key].to_s, "\n"
        end
      end
    end
  end
```

* simple_router.rb
```
  def show_db
    return @routing_table.db
  end
```

* routing_table.rb
```
  attr_accessor :db
```

経路表が格納されているrouting_table.rb内のdbにアクセスするためにattr_accessor :dbを用いることによって実現した。

###ルーティングテーブルエントリの追加と削除
書き加えたコードを以下に示す。

* routing_table
```
  desc 'add forwardind entry'
  arg_name 'dest netmask next'
  command :add do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      add_db_entry(args[0], args[1], args[2])
    end
  end

  desc 'delete forwardind entry'
  arg_name 'dest netmask'
  command :delete do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      delete_db_entry(args[0], args[1], nil)
    end
  end
```

* simple_router.rb
```
  def add_db_entry(destination_ip, netmask_length, next_hop)
    options = {:destination => destination_ip, :netmask_length => netmask_length.to_i, :next_hop => next_hop}
    @routing_table.add(options)
  end

  def delete_db_entry(destination_ip, netmask_length, next_hop)
    options = {:destination => destination_ip, :netmask_length => netmask_length.to_i, :next_hop => next_hop}
    @routing_table.delete(options)
  end
```

* routing_table.rb
```
  def delete(options)
    netmask_length = options.fetch(:netmask_length)
    prefix = IPv4Address.new(options.fetch(:destination)).mask(netmask_length)
    @db[netmask_length].delete(prefix.to_i)
  end
```

追加はrouting_table.rbに記載されていたaddを用いた。削除についてもaddと同じように作成した。

###ルータのインターフェース一覧の表示
書き加えたコードを以下に示す。

* routing_table
```
  desc 'show interfaces'
  command :show_interfaces do |c|
    c.desc 'Location to find socket files'
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR
    c.action do |_global_options, options, args|
      interfaces = Trema.trema_process('SimpleRouter', options[:socket_dir]).controller.
      show_interface
      interfaces.each do |each|
        print "port : ", each[:port_number].to_s, ", MAC : ", each[:mac_address], ", IP : ", each[:ip_address], "/", each[:netmask_length].to_s, "\n"
      end
    end
  end
```

* simple_router.rb
```
  def show_interface
    ret = Array.new()
    Interface.all.each do |each|
      ret.push(:port_number => each.port_number, :mac_address => each.mac_address.to_s, :ip_address => each.ip_address.value.to_s, :netmask_length => each.netmask_length)
    end
    return ret
  end
```

retという変数を用いて、Interfaceの要素を格納している。この変数をrouting_tableに返すことによって動作するようになっている。

###動作確認
```
$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Nagatomi-Ken$ ./bin/routing_table show_db
routing table
Dest : 0.0.0.0/0
Next : 192.168.1.2

$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Nagatomi-Ken$ ./bin/routing_table add 192.168.3.2 24 192.168.2.2
$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Nagatomi-Ken$ ./bin/routing_table show_db
routing table
Dest : 0.0.0.0/0
Next : 192.168.1.2
Dest : 192.168.3.2/24
Next : 192.168.2.2

$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Nagatomi-Ken$ ./bin/routing_table delete 192.168.3.2 24
$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Nagatomi-Ken$ ./bin/routing_table show_db
routing table
Dest : 0.0.0.0/0
Next : 192.168.1.2

$ ensyuu2@ensyuu2-VirtualBox:~/simple-router-Shu-NISHIKORI$ ./bin/routing_table show_interface
port : 1, MAC : 01:01:01:01:01:01, IP : 192.168.1.1/24
port : 2, MAC : 02:02:02:02:02:02, IP : 192.168.2.1/24
```
正しく動作していることが確認できた。


