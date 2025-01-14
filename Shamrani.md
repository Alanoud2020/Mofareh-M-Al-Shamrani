# Mofareh-M-Al-Shamrani
SPSS
#include<accounting/personnel>. [Descriptive variable statistics](https://user-images.githubusercontent.com/95216143/143929169-9bdfeb58-401d-4ce6-b619-86f003c4bdb0.png)

#include<Chart> ![DVM](https://user-images.githubusercontent.com/95216143/143930879-c7101ff4-346a-4d6e-bed5-84057cf6f998.png)


#include<![chart](https://user-images.githubusercontent.com/95216143/143929130-c5771d83-22f2-45e6-80ba-c07b6ae6e87b.png)

#include<corelation/matrix>  ![matrix](https://user-images.githubusercontent.com/95216143/143929542-33b15a6a-810c-4550-9935-394d8bf2d222.png)

#include<regression/matrix>  ![regression](https://user-images.githubusercontent.com/95216143/143929617-591cc954-4028-42e6-9703-4d7695217d9e.png)


Include<formulaused>.  ![formula](https://user-images.githubusercontent.com/95216143/143929673-39e46dc6-16fb-4593-909a-ab6078a0960c.png)
  /* driver name */
#define DEVICE_NAME	"spss-utils"

enum spss_firmware_type {
	SPSS_FW_TYPE_DEV = 'd',
	SPSS_FW_TYPE_TEST = 't',
	SPSS_FW_TYPE_PROD = 'p',
	SPSS_FW_TYPE_NONE = 'z',
};

static enum spss_firmware_type firmware_type = SPSS_FW_TYPE_TEST;
static const char *dev_firmware_name;
static const char *test_firmware_name;
static const char *prod_firmware_name;
static const char *none_firmware_name = "nospss";
static const char *firmware_name = "NA";
static struct device *spss_dev;
static u32 spss_debug_reg_addr; /* SP_SCSR_MBn_SP2CL_GPm(n,m) */
static u32 spss_emul_type_reg_addr; /* TCSR_SOC_EMULATION_TYPE */

#define SPU_EMULATUION (BIT(0) | BIT(1))
#define SPU_PRESENT_IN_EMULATION BIT(2)

/*==========================================================================*/
/*		Device Sysfs */
/*==========================================================================*/

static ssize_t firmware_name_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
	int ret;

	if (!dev || !attr || !buf) {
		pr_err("invalid param.\n");
		return -EINVAL;
	}

	if (firmware_name == NULL)
		ret = snprintf(buf, PAGE_SIZE, "%s\n", "unknown");
	else
		ret = snprintf(buf, PAGE_SIZE, "%s\n", firmware_name);

	return ret;
}

static ssize_t firmware_name_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf,
		size_t size)
{
	pr_err("set firmware name is not allowed.\n");

	return -EINVAL;
}

static DEVICE_ATTR(firmware_name, 0444,
		firmware_name_show, firmware_name_store);

static ssize_t test_fuse_state_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
	int ret;

	if (!dev || !attr || !buf) {
		pr_err("invalid param.\n");
		return -EINVAL;
	}

	switch (firmware_type) {
	case SPSS_FW_TYPE_DEV:
		ret = snprintf(buf, PAGE_SIZE, "%s", "dev");
		break;
	case SPSS_FW_TYPE_TEST:
		ret = snprintf(buf, PAGE_SIZE, "%s", "test");
		break;
	case SPSS_FW_TYPE_PROD:
		ret = snprintf(buf, PAGE_SIZE, "%s", "prod");
		break;
	default:
		return -EINVAL;
	}

	return ret;
}

static ssize_t test_fuse_state_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf,
		size_t size)
{
	pr_err("set test fuse state is not allowed.\n");

	return -EINVAL;
}

static DEVICE_ATTR(test_fuse_state, 0444,
		test_fuse_state_show, test_fuse_state_store);

static ssize_t spss_debug_reg_show(struct device *dev,
		struct device_attribute *attr,
		char *buf)
{
	int ret;
	void __iomem *spss_debug_reg = NULL;
	u32 val1, val2;

	if (!dev || !attr || !buf) {
		pr_err("invalid param.\n");
		return -EINVAL;
	}

	pr_debug("spss_debug_reg_addr [0x%x].\n", spss_debug_reg_addr);

	spss_debug_reg = ioremap_nocache(spss_debug_reg_addr, sizeof(u32)*2);

	if (!spss_debug_reg) {
		pr_err("can't map debug reg addr.\n");
		return -EFAULT;
	}

	val1 = readl_relaxed(spss_debug_reg);
	val2 = readl_relaxed(((char *) spss_debug_reg) + sizeof(u32));

	ret = snprintf(buf, PAGE_SIZE, "val1 [0x%x] val2 [0x%x]", val1, val2);

	iounmap(spss_debug_reg);

	return ret;
}

static ssize_t spss_debug_reg_store(struct device *dev,
		struct device_attribute *attr,
		const char *buf,
		size_t size)
{
	pr_err("set debug reg is not allowed.\n");

	return -EINVAL;
}

static DEVICE_ATTR(spss_debug_reg, 0444,
		spss_debug_reg_show, spss_debug_reg_store);

static int spss_create_sysfs(struct device *dev)
{
	int ret;

	ret = device_create_file(dev, &dev_attr_firmware_name);
	if (ret < 0) {
		pr_err("failed to create sysfs file for firmware_name.\n");
		return ret;
	}

	ret = device_create_file(dev, &dev_attr_test_fuse_state);
	if (ret < 0) {
		pr_err("failed to create sysfs file for test_fuse_state.\n");
		device_remove_file(dev, &dev_attr_firmware_name);
		return ret;
	}

	ret = device_create_file(dev, &dev_attr_spss_debug_reg);
	if (ret < 0) {
		pr_err("failed to create sysfs file for spss_debug_reg.\n");
		device_remove_file(dev, &dev_attr_firmware_name);
		device_remove_file(dev, &dev_attr_test_fuse_state);
		return ret;
	}

	return 0;
}

/*==========================================================================*/
/*		Device Tree */
/*==========================================================================*/

/**
 * spss_parse_dt() - Parse Device Tree info.
 */
static int spss_parse_dt(struct device_node *node)
{
	int ret;
	u32 spss_fuse1_addr = 0;
	u32 spss_fuse1_bit = 0;
	u32 spss_fuse1_mask = 0;
	void __iomem *spss_fuse1_reg = NULL;
	u32 spss_fuse2_addr = 0;
	u32 spss_fuse2_bit = 0;
	u32 spss_fuse2_mask = 0;
	void __iomem *spss_fuse2_reg = NULL;
	u32 val1 = 0;
	u32 val2 = 0;
	void __iomem *spss_emul_type_reg = NULL;
	u32 spss_emul_type_val = 0;

	ret = of_property_read_string(node, "qcom,spss-dev-firmware-name",
		&dev_firmware_name);
	if (ret < 0) {
		pr_err("can't get dev fw name.\n");
		return -EFAULT;
	}

	ret = of_property_read_string(node, "qcom,spss-test-firmware-name",
		&test_firmware_name);
	if (ret < 0) {
		pr_err("can't get test fw name.\n");
		return -EFAULT;
	}

	ret = of_property_read_string(node, "qcom,spss-prod-firmware-name",
		&prod_firmware_name);
	if (ret < 0) {
		pr_err("can't get prod fw name.\n");
		return -EFAULT;
	}

	ret = of_property_read_u32(node, "qcom,spss-fuse1-addr",
		&spss_fuse1_addr);
	if (ret < 0) {
		pr_err("can't get fuse1 addr.\n");
		return -EFAULT;
	}

	ret = of_property_read_u32(node, "qcom,spss-fuse2-addr",
		&spss_fuse2_addr);
	if (ret < 0) {
		pr_err("can't get fuse2 addr.\n");
		return -EFAULT;
	}

	ret = of_property_read_u32(node, "qcom,spss-fuse1-bit",
		&spss_fuse1_bit);
	if (ret < 0) {
		pr_err("can't get fuse1 bit.\n");
		return -EFAULT;
	}

	ret = of_property_read_u32(node, "qcom,spss-fuse2-bit",
		&spss_fuse2_bit);
	if (ret < 0) {
		pr_err("can't get fuse2 bit.\n");
		return -EFAULT;
	}


	spss_fuse1_mask = BIT(spss_fuse1_bit);
	spss_fuse2_mask = BIT(spss_fuse2_bit);

	pr_debug("spss fuse1 addr [0x%x] bit [%d] .\n",
		(int) spss_fuse1_addr, (int) spss_fuse1_bit);
	pr_debug("spss fuse2 addr [0x%x] bit [%d] .\n",
		(int) spss_fuse2_addr, (int) spss_fuse2_bit);

	spss_fuse1_reg = ioremap_nocache(spss_fuse1_addr, sizeof(u32));

	if (!spss_fuse1_reg) {
		pr_err("can't map fuse1 addr.\n");
		return -EFAULT;
	}

	spss_fuse2_reg = ioremap_nocache(spss_fuse2_addr, sizeof(u32));

	if (!spss_fuse2_reg) {
		iounmap(spss_fuse1_reg);
		pr_err("can't map fuse2 addr.\n");
		return -EFAULT;
	}

	val1 = readl_relaxed(spss_fuse1_reg);
	val2 = readl_relaxed(spss_fuse2_reg);

	pr_debug("spss fuse1 value [0x%08x].\n", (int) val1);
	pr_debug("spss fuse2 value [0x%08x].\n", (int) val2);

	pr_debug("spss fuse1 mask [0x%08x].\n", (int) spss_fuse1_mask);
	pr_debug("spss fuse2 mask [0x%08x].\n", (int) spss_fuse2_mask);

	/**
	 * Set firmware_type based on fuses:
	 *	SPSS_CONFIG_MODE 11:        dev
	 *	SPSS_CONFIG_MODE 01 or 10:  test
	 *	SPSS_CONFIG_MODE 00:        prod
	 */
	if ((val1 & spss_fuse1_mask) && (val2 & spss_fuse2_mask))
		firmware_type = SPSS_FW_TYPE_DEV;
	else if ((val1 & spss_fuse1_mask) || (val2 & spss_fuse2_mask))
		firmware_type = SPSS_FW_TYPE_TEST;
	else
		firmware_type = SPSS_FW_TYPE_PROD;

	iounmap(spss_fuse1_reg);
	iounmap(spss_fuse2_reg);

	ret = of_property_read_u32(node, "qcom,spss-debug-reg-addr",
		&spss_debug_reg_addr);
	if (ret < 0) {
		pr_err("can't get debug regs addr.\n");
		return ret;
	}

	ret = of_property_read_u32(node, "qcom,spss-emul-type-reg-addr",
			     &spss_emul_type_reg_addr);
	if (ret < 0) {
		pr_err("can't get spss-emulation-type-reg addr\n");
		return -EFAULT;
	}

	spss_emul_type_reg = ioremap_nocache(spss_emul_type_reg_addr,
					     sizeof(u32));
	if (!spss_emul_type_reg) {
		iounmap(spss_emul_type_reg);
		pr_err("can't map soc-emulation-type reg addr.\n");
		return -EFAULT;
	}

	spss_emul_type_val = readl_relaxed(spss_emul_type_reg);

	pr_debug("spss_emul_type value [0x%08x].\n", (int)spss_emul_type_val);
	if ((spss_emul_type_val & SPU_EMULATUION) &&
	    !(spss_emul_type_val & SPU_PRESENT_IN_EMULATION)) {
		/* for some emulation platforms SPSS is not present */
		firmware_type = SPSS_FW_TYPE_NONE;
	}
	iounmap(spss_emul_type_reg);

	return 0;
}

/**
 * spss_probe() - initialization sequence
 */
static int spss_probe(struct platform_device *pdev)
{
	int ret = 0;
	struct device_node *np = NULL;
	struct device *dev = NULL;

	if (!pdev) {
		pr_err("invalid pdev.\n");
		return -ENODEV;
	}

	np = pdev->dev.of_node;
	if (!np) {
		pr_err("invalid DT node.\n");
		return -EINVAL;
	}

	dev = &pdev->dev;
	spss_dev = dev;

	if (dev == NULL) {
		pr_err("invalid dev.\n");
		return -EINVAL;
	}

	platform_set_drvdata(pdev, dev);

	ret = spss_parse_dt(np);
	if (ret < 0) {
		pr_err("fail to parse device tree.\n");
		return -EFAULT;
	}

	switch (firmware_type) {
	case SPSS_FW_TYPE_DEV:
		firmware_name = dev_firmware_name;
		break;
	case SPSS_FW_TYPE_TEST:
		firmware_name = test_firmware_name;
		break;
	case SPSS_FW_TYPE_PROD:
		firmware_name = prod_firmware_name;
		break;
	case SPSS_FW_TYPE_NONE:
		firmware_name = none_firmware_name;
		break;
	default:
		return -EINVAL;
	}

	ret = subsystem_set_fwname("spss", firmware_name);
	if (ret < 0) {
		pr_err("fail to set fw name.\n");
		return -EFAULT;
	}

	ret = spss_create_sysfs(dev);
	if (ret < 0) {
		pr_err("fail to create sysfs.\n");
		return -EFAULT;
	}

	pr_info("Initialization completed ok, firmware_name [%s].\n",
		firmware_name);

	return 0;
}

static const struct of_device_id spss_match_table[] = {
	{ .compatible = "qcom,spss-utils", },
	{ },
};

static struct platform_driver spss_driver = {
	.probe = spss_probe,
	.driver = {
		.name = DEVICE_NAME,
		.owner = THIS_MODULE,
		.of_match_table = of_match_ptr(spss_match_table),
	},
};

/*==========================================================================*/
/*		Driver Init/Exit					*/
/*==========================================================================*/
static int __init spss_init(void)
{
	int ret = 0;

	pr_info("spss-utils driver Ver 3.0 18-Feb-2018.\n");

	ret = platform_driver_register(&spss_driver);
	if (ret)
		pr_err("register platform driver failed, ret [%d]\n", ret);

	return ret;
}
late_initcall(spss_init); /* start after PIL driver */

MODULE_LICENSE("GPL v2");
MODULE_DESCRIPTION("Secure Processor Utilities");

  * TURN OFF SCIENTIFIC NOTATION

 SET DIGITGROUPING=No Small=0 Unicode=No OLang=English.

 * OPEN THE DATASET

 GET 
   FILE='[Zezo.xlsx](https://github.com/LFASEWWSVDS/Mofareh-M-Al-Shamrani/files/7622697/Zezo.xlsx)'. 

 DATASET NAME WINDOW=FRONT.
 DATASET ACTIVATE.

 *Test the 5 screening criteria with Frequencies.

 FREQUENCIES VARIABLES=NPM, AS, AL, AUI        
   /FORMAT=NOTABLE
   /STATISTICS=RANGE MEAN
   /ORDER=ANALYSIS.

 *Test for non-linearity with scatterplot for �Had Crying Spells� (it has large skewness and Kurtosis scores) against the 24 other items being tested.  

 REGRESSION
   /MISSING LISTWISE
   /STATISTICS COEFF OUTS R ANOVA
   /CRITERIA=PIN(.05) POUT(.10)
   /NOORIGIN 
   /DEPENDENT cesd10c
   /METHOD=ENTER NPM, AS, AL, AUI
   /PARTIALPLOT ALL.

 * Test for influential cases and multicollinearity 

 REGRESSION
   /MISSING LISTWISE
   /STATISTICS ALL
   /CRITERIA=PIN(.05) POUT(.10)
   /NOORIGIN
   /DEPENDENT id 
   /METHOD=ENTER 
   /SAVE MAHAL.


 * Run 1 factor EFA for the 8 Happy and Sad items of the CESD

 FACTOR VARIABLES=NPM, AS, AL, AUI
   /MISSING LISTWISE
   /ANALYSIS NPM, AS, AL, AUI
   /PRINT INITIAL UNIVARIATE CORRELATION ROTATION REPR
   /FORMAT SORT BLANK (.30)
   /PLOT EIGEN
   /CRITERIA FACTORS(1) ITERATE(25)
   /EXTRACTION PAF
   /ROTATION OBLIMIN
   /METHOD=CORRELATION.

 * CLOSE THE DATASET

 DATASET CLOSE Homework5.

 * Export Output.
 OUTPUT EXPORT
   /CONTENTS  EXPORT=ALL  LAYERS=ALL  MODELVIEWS=ALL
   /PDF  DOCUMENTFILE='[My Thesis.pdf](https://github.com/LFASEWWSVDS/Mofareh-M-Al-Shamrani/files/7622720/My.Thesis.pdf)
'+
      EMBEDBOOKMARKS=YES  EMBEDFONTS=YES.
  - Thank you
