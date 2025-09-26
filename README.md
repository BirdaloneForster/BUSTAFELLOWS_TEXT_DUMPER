# BUSTAFELLOWS_TEXT_DUMPER
我想要把switch版的BUSTAFELLOWS官中移植到PC Steam版上，这是逆向分析得到的解码游戏文本的脚本。

# 原相关代码
## 读取本地二进制数据部分代码
```c#
AssetLoader.binReader = new UniScript.Decoder.Sk2.BinReader();
AssetLoader.binReader.Ready(AssetLoader.ReadBytes("sk_ep0_script_2.bytes"));
		yield return null;
		AssetLoader.textTable = new TextTableReader();
		AssetLoader.textTable.Ready(AssetLoader.ReadBytes("sk_ep0_text_2.bytes"), "游戏字符表，这里省略了，太长了，不过做了繁中字符");
		GlobalData.AlreadyRead.Length = AssetLoader.textTable.Length + 1;
```
## 继续看TextTableReader这里是啥

```c#
public class TextTableReader : BinaryDataReader, ITextSource
	{
		protected override byte[] Magic
		{
			get
			{
				return TextTableReader.magic;
			}
		}
		protected override byte[] TableEnd
		{
			get
			{
				return TextTableReader.tableEnd;
			}
		}
		private string CharTable { get; set; }
		public int Length
		{
			get
			{
				return this.table.GetLength(0);
			}
		}
		public override bool Ready(byte[] data)
		{
			return base.Ready(data);
		}
		public bool Ready(byte[] data, string charTable)
		{
			this.CharTable = charTable;
			return base.Ready(data);
		}
		public string GetText(int index)
		{
			index--;
			bool flag = index < 0;
			string text;
			if (flag)
			{
				text = null;
			}
			else
			{
				this.ptr = this.table[index, 0];
				int num = this.ptr + this.table[index, 1];
				this.sb.Length = 0;
				while (this.ptr < num)
				{
					int num2 = base.ReadUnsigned();
					bool flag2 = num2 == 0;
					if (flag2)
					{
						this.sb.Append('\n');
					}
					else
					{
						this.sb.Append(this.CharTable[num2 - 1]);
					}
				}
				text = this.sb.ToString();
			}
			return text;
		}
		private static readonly byte[] magic = new byte[] { 85, 84, 88, 84 };
		private static readonly byte[] tableEnd = new byte[] { 84, 69, 78, 68 };
		private readonly StringBuilder sb = new StringBuilder();
	}
```
## 继续看BinaryDataReader这里是啥

```c#
public abstract class BinaryDataReader : MultiBytesReader
	{
		public int Version { get; private set; }
		public DateTime CreationTime { get; private set; }
		protected abstract byte[] Magic { get; }
		protected abstract byte[] TableEnd { get; }
		public override bool Ready(byte[] bytes)
		{
			base.Ready(bytes);
			bool flag = !this.IsSequential(this.Magic);
			bool flag2;
			if (flag)
			{
				flag2 = false;
			}
			else
			{
				this.Version = this.ReadPlainInt();
				this.CreationTime = this.ReadDateTime();
				int num = this.ReadPlainInt();
				int[] array = this.ReadUnsignedArray();
				bool flag3 = !this.IsSequential(this.TableEnd);
				if (flag3)
				{
					flag2 = false;
				}
				else
				{
					bool flag4 = this.ptr != num;
					if (flag4)
					{
						flag2 = false;
					}
					else
					{
						this.table = new int[array.Length, 2];
						int num2 = 0;
						for (int i = 0; i < array.Length; i++)
						{
							this.table[i, 0] = num2;
							this.table[i, 1] = array[i];
							num2 += array[i];
						}
						byte[] array2 = new byte[bytes.Length - this.ptr];
						Array.Copy(bytes, this.ptr, array2, 0, array2.Length);
						flag2 = base.Ready(array2);
					}
				}
			}
			return flag2;
		}
		protected bool IsSequential(byte[] referTo)
		{
			foreach (byte b in referTo)
			{
				byte b2 = b;
				byte[] data = this.data;
				int ptr = this.ptr;
				this.ptr = ptr + 1;
				bool flag = b2 != data[ptr];
				if (flag)
				{
					return false;
				}
			}
			return true;
		}
		protected int ReadPlainInt()
		{
			byte[] data = this.data;
			int num = this.ptr;
			this.ptr = num + 1;
			int num2 = data[num];
			int num3 = num2;
			byte[] data2 = this.data;
			num = this.ptr;
			this.ptr = num + 1;
			num2 = num3 | (data2[num] << 8);
			int num4 = num2;
			byte[] data3 = this.data;
			num = this.ptr;
			this.ptr = num + 1;
			num2 = num4 | (data3[num] << 16);
			int num5 = num2;
			byte[] data4 = this.data;
			num = this.ptr;
			this.ptr = num + 1;
			return num5 | (data4[num] << 24);
		}
		protected DateTime ReadDateTime()
		{
			int num = this.ReadPlainInt();
			return new DateTime(2018, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc).AddMinutes((double)num);
		}
		protected int[] ReadUnsignedArray()
		{
			int num = base.ReadUnsigned();
			int[] array = new int[num];
			for (int i = 0; i < array.Length; i++)
			{
				array[i] = base.ReadUnsigned();
			}
			return array;
		}
		protected int[,] table;
	}
```



## 继续看MultiBytesReader这里是啥

```c#
public class MultiBytesReader
	{
		public static int ErrorValue { get; } = int.MinValue;
		public byte[] CloneData
		{
			get
			{
				byte[] array = new byte[this.data.Length];
				Array.Copy(this.data, array, this.data.Length);
				return array;
			}
		}
		public int CurrentPosition
		{
			get
			{
				return this.ptr;
			}
		}
		public void ResetPosition()
		{
			this.ptr = 0;
		}
		public virtual bool Ready(byte[] data)
		{
			this.data = data;
			this.ptr = 0;
			return true;
		}
		public int ReadSigned()
		{
			byte[] array = this.data;
			int num = this.ptr;
			this.ptr = num + 1;
			int num2 = array[num];
			bool flag = (num2 & 128) == 0;
			int num3;
			if (flag)
			{
				num3 = (num2 & 127) << 25 >> 25;
			}
			else
			{
				byte[] array2 = this.data;
				num = this.ptr;
				this.ptr = num + 1;
				int num4 = array2[num];
				bool flag2 = (num4 & 128) == 0;
				if (flag2)
				{
					num3 = (((num2 & 127) << 7) | (num4 & 127)) << 18 >> 18;
				}
				else
				{
					byte[] array3 = this.data;
					num = this.ptr;
					this.ptr = num + 1;
					int num5 = array3[num];
					bool flag3 = (num5 & 128) == 0;
					if (flag3)
					{
						num3 = (((num2 & 127) << 14) | ((num4 & 127) << 7) | (num5 & 127)) << 11 >> 11;
					}
					else
					{
						byte[] array4 = this.data;
						num = this.ptr;
						this.ptr = num + 1;
						int num6 = array4[num];
						bool flag4 = (num6 & 128) == 0;
						if (flag4)
						{
							num3 = (((num2 & 127) << 21) | ((num4 & 127) << 14) | ((num5 & 127) << 7) | (num6 & 127)) << 4 >> 4;
						}
						else
						{
							num3 = MultiBytesReader.ErrorValue;
						}
					}
				}
			}
			return num3;
		}

		// Token: 0x06000038 RID: 56 RVA: 0x00002E74 File Offset: 0x00001074
		public int ReadUnsigned()
		{
			byte[] array = this.data;
			int num = this.ptr;
			this.ptr = num + 1;
			int num2 = array[num];
			bool flag = (num2 & 128) == 0;
			int num3;
			if (flag)
			{
				num3 = num2 & 127;
			}
			else
			{
				byte[] array2 = this.data;
				num = this.ptr;
				this.ptr = num + 1;
				int num4 = array2[num];
				bool flag2 = (num4 & 128) == 0;
				if (flag2)
				{
					num3 = ((num2 & 127) << 7) | (num4 & 127);
				}
				else
				{
					byte[] array3 = this.data;
					num = this.ptr;
					this.ptr = num + 1;
					int num5 = array3[num];
					bool flag3 = (num5 & 128) == 0;
					if (flag3)
					{
						num3 = ((num2 & 127) << 14) | ((num4 & 127) << 7) | (num5 & 127);
					}
					else
					{
						byte[] array4 = this.data;
						num = this.ptr;
						this.ptr = num + 1;
						int num6 = array4[num];
						bool flag4 = (num6 & 128) == 0;
						if (flag4)
						{
							num3 = ((num2 & 127) << 21) | ((num4 & 127) << 14) | ((num5 & 127) << 7) | (num6 & 127);
						}
						else
						{
							num3 = MultiBytesReader.ErrorValue;
						}
					}
				}
			}
			return num3;
		}
		protected byte[] data;
		protected int ptr;
	}
```

